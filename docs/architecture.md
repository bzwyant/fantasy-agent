# Fantasy Agent – Backend-First Architecture

## Purpose

An AI-powered fantasy football backend that analyzes rosters, evaluates trades, prioritizes waivers, and surfaces insights by aggregating data from fantasy platforms, news, social, and stats. Client is minimal initially; focus is on robust, scalable server services.

## High-Level Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                              Client (Vercel)                            │
│  - Next.js app (minimal UI)                                            │
│  - Clerk authentication (Hosted pages or minimal components)            │
│  - Calls server APIs with Bearer Clerk JWT                              │
└────────────────────────────────────────────────────────────────────────┘
                     │ HTTPS (JWT: Clerk, audience: api)
                     ▼
┌────────────────────────────────────────────────────────────────────────┐
│                               API Layer (AWS)                           │
│  - Amazon API Gateway (JWT authorizer: Clerk JWKS)                      │
│  - Routes proxy to Lambda handlers (SST)                                │
└────────────────────────────────────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────────────────┐
│                     Application + Domain (SST on AWS)                   │
│  - Application (use cases, orchestration)                               │
│  - Domain (entities, value objects, services)                           │
│  - Integrations (HTTP clients: ESPN/Yahoo/Sleeper, Reddit/Twitter, etc) │
│  - Analysis pipelines (LLM calls, scoring, projections)                 │
│  - Messaging (SQS) & Schedulers (EventBridge/Cron)                      │
│  - Caching (ElastiCache/Redis)                                          │
│  - Storage (DynamoDB for state/cache, RDS/ClickHouse for analytics)     │
└────────────────────────────────────────────────────────────────────────┘
```

## Technology Choices

- Client: Next.js on Vercel, Clerk for auth (JWT), minimal UI to start
- Server: SST v3 on AWS (Lambda, API Gateway, SQS, EventBridge, DynamoDB, RDS/ClickHouse, ElastiCache)
- AI: OpenAI/Anthropic via server-side integrations
- Optional tools: MCP (Model Context Protocol) later; start with internal HTTP services

## Authentication and Authorization

- Identity Provider: Clerk
- Client obtains Clerk session/JWT and sends `Authorization: Bearer <JWT>` to the API
- API Gateway JWT Authorizer validates against Clerk JWKS
  - issuer: `https://<your-clerk-domain>/.well-known/jwks.json`
  - audience: `fantasy-agent-api`
- Lambda receives verified claims for RBAC/ownership checks

## Data Domains (Core)

- Players, Teams, Rosters, Leagues, Matchups
- Analysis: WeeklyAnalysis, TradeEvaluation, WaiverRecommendation
- Market/News: NewsItem, Sentiment, Trends

## Storage Strategy

- DynamoDB: operational state, denormalized entity snapshots, analysis cache with TTL
- Redis (ElastiCache): shared cache for hot keys (players, lineups, projections)
- RDS or ClickHouse: analytics, time-series queries, evaluation history
- S3: cold storage/exports and large artifacts

## Caching Strategy (simple policy)

- Redis as primary shared cache (TTL by domain: projections 15m, news 5m, roster 1m)
- DynamoDB cache table with TTL for computed analysis (per team/week)
- Avoid assuming Lambda memory cache for correctness; treat only as per-invocation optimization

## Background Workflows

- Scheduled jobs (EventBridge Cron): weekly analysis, player syncs, news processing
- Queues (SQS): fan-out work (per-team analysis, trade eval batches)
- Idempotency keys on jobs; retries with backoff; DLQ for failures

## Integrations

- Fantasy Platforms: ESPN, Yahoo, Sleeper (HTTP clients with per-provider rate limits)
- News/Social: Reddit/Twitter, RSS feeds (batched + cached)
- AI Providers: OpenAI/Anthropic (budget caps, tracing, retries)

## Optional: MCP (Model Context Protocol)

- What it is: a protocol that exposes “tools” and “resources” to LLM clients via JSON-RPC (stdio/websocket)
- When to use: if you want external LLM clients (e.g., Claude Desktop) to directly discover/call your tools
- Default approach here: keep internal services as HTTP microservices behind API Gateway; add MCP later only if needed

## Observability

- Logging: structured JSON to CloudWatch; correlation IDs propagated through layers
- Metrics: latency, success rate, cache hit rate, cost per request
- Tracing: OpenTelemetry (optional vendor: Datadog/Honeycomb); capture LLM tokens/costs (Langfuse optional)

## Security

- JWT verification at API Gateway (Clerk JWKS)
- KMS for secrets; no secrets in logs
- Provider-specific rate limits + circuit breakers
- Least-privilege IAM for Lambdas and data stores

## Scalability & Cost

- Lambda with provisioned concurrency on latency-critical handlers; scale-to-zero elsewhere
- On-demand DynamoDB; Redis sized for hotset
- Batch external API calls; coalesce identical requests
- Consider ClickHouse/Redshift Serverless if analytics scale out

## Project Structure (Monorepo)

```
fantasy-agent/
├── apps/
│   ├── client/                         # Next.js app (Vercel)
│   │   ├── app/                        # App Router (routes + server actions)
│   │   ├── components/
│   │   ├── lib/                        # API client, Clerk helpers
│   │   ├── public/
│   │   ├── styles/
│   │   └── next.config.js
│   └── server/                         # SST app (AWS)
│       ├── sst.config.ts               # Infra definitions (API, Functions, Queues, DB)
│       └── src/
│           ├── application/            # Use cases, orchestrators
│           ├── domain/                 # Entities, value-objects, domain services
│           ├── infrastructure/
│           │   ├── gateways/           # HTTP clients for external providers
│           │   ├── repositories/       # DynamoDB/RDS implementations
│           │   ├── messaging/          # SQS producers/consumers
│           │   ├── schedulers/         # EventBridge cron handlers
│           │   └── http/               # Lambda HTTP handlers (API routes)
│           ├── shared/                 # Cross-cutting (config, logging, errors)
│           └── workers/                # Long-running or batch tasks
├── packages/
│   ├── interfaces/                     # Shared TypeScript interfaces
│   │   ├── domain/                     # Player, Team, Roster, etc.
│   │   ├── analysis/                   # WeeklyAnalysis, TradeEvaluation, WaiverRecommendation
│   │   └── api/                        # Request/Response contracts between frontend/backend
│   └── utils/                          # Shared utilities (types, validation, schema)
└── docs/
    └── architecture.md                 # This document
```

## API Surfaces (concise)

- POST `/agent/analyze-weekly` → returns `WeeklyAnalysis`
- POST `/agent/evaluate-trade` → returns `TradeEvaluation`
- GET `/teams/:teamId/roster` → returns `Roster`
- GET `/players/:playerId` → returns `Player`
- All endpoints require Clerk JWT (audience `fantasy-agent-api`)

## Deployment Model

- Client: Vercel project `fantasy-client` (Clerk Next SDK)
- Server: SST deploys API Gateway + Lambdas + data stores in `us-east-1`
- CI/CD: Github Actions → SST deploy (dev/prod); Vercel integrations for client

## Minimal Getting Started (no code, just steps)

1. Create Clerk app and configure JWT template (audience `fantasy-agent-api`)
2. Bootstrap SST server: API Gateway (JWT authorizer using Clerk JWKS), core Lambdas, DynamoDB tables, Redis, analytics store
3. Create Next.js app on Vercel with Clerk; call server with Bearer JWT
4. Add scheduled jobs (weekly analysis, player sync) and SQS workers
5. Add caching TTLs and rate limits per provider; enable tracing/logging

---

This document intentionally omits heavy implementation code. It defines the target architecture, security model (Clerk JWTs), service boundaries, storage/caching strategy, and a monorepo structure with shared interfaces to build a backend-first fantasy agent that scales.
