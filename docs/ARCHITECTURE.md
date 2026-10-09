# STRATUM Architecture (Public)

This is the public architecture overview. Operational runbook details, security configuration specifics, and internal invariants are kept in the private repos.

## System overview

STRATUM collects financial market data weekly, processes it through a medallion pipeline (raw → bronze → silver → gold), serves the results through a REST API and an ops UI, and adds an AI layer on Amazon Bedrock: a read-only **Ops Chat** and a human-gated **change pipeline** that turns a change request into a tested pull request. The primary design constraint is **cost**: no ALB, no NAT Gateway, no RDS instances, compute on Fargate Spot, and automated shutdown outside working hours. Deployed in us-east-1, entirely via Terraform (HCP Terraform), with branch → environment promotion: `develop → DEV`, `release → UAT`, `master → PROD` (only DEV is deployed).

| Component | What it is |
|-----------|-----------|
| Data pipeline | EventBridge → Step Functions → Lambda ingestion + three Glue jobs, Parquet on S3 |
| Gateway | One FastAPI service on ECS Fargate Spot: data API, ops actions, access management, Ops Chat, change pipeline |
| Ops UI | Next.js on Amplify, Google SSO via Cognito, calls the gateway with the signed-in user's token |
| AI layer | Claude Sonnet 4.6 on Amazon Bedrock, usage and cost recorded for every call |
| Data store | DynamoDB (pipeline runs, access requests) and serverless Aurora DSQL (AI audit, chat, jobs) |

## Data pipeline

An EventBridge rule fires weekly and starts a Step Functions state machine that:

1. Records the run in DynamoDB (status RUNNING) and sends an SNS start notification (best-effort — a missed email is not a reason to skip the run).
2. Invokes the **data-ingestor Lambda**, which fetches market data (US tech, Indian NSE equities, crypto, indices, commodities) from the Yahoo Finance chart API via direct HTTP and writes one JSON file per symbol to the S3 raw bucket under a date-partitioned prefix.
3. Runs three Glue jobs sequentially using `.sync` integration: **raw→bronze** (schema enforcement, dedup), **bronze→silver** (business transforms, enrichment), **silver→gold** (aggregations and KPIs: weekly summary, top/worst performers, volatility report, asset-class summary, per-symbol detail). All output is Parquet.
4. Marks the run SUCCEEDED/FAILED in DynamoDB and sends a completion/failure notification.

The **gold layer is the serving layer** — the gateway reads it directly from S3, resolving the latest `ingested_date` partition at query time. There is no database between the pipeline and the data API.

## Gateway and ops UI

The gateway is a FastAPI service on ECS Fargate Spot in public subnets. There is no load balancer: CloudFront points at a Route53 A record holding the current task's public IP. When Spot replaces the task, an EventBridge rule on ECS Task State Change events triggers a Lambda that reads the new ENI's public IP and upserts the A record. The Lambda retries internally because the IP isn't always assigned the instant ECS reports RUNNING.

The API is OpenAPI-documented (Swagger UI) and grouped by purpose: data and pipeline reads, explicit admin-only **ops actions** (run the ETL pipeline, stop/start/restore ECS services), access management, and the AI routes. The **ops UI** (Amplify) gives the same platform a front end: chat, status and budget, approvals and users, an admin view of AI usage, and the Change Pipeline tab.

## Auth

- **Humans only.** Google SSO through Cognito; no machine-to-API credentials.
- Every request carries a Cognito **access token**. One FastAPI dependency verifies the signature (cached JWKS), issuer, expiry, token type and client.
- **Two platform roles** — `stratum-user` and `stratum-admin` — shared by every app. Authorization is declared per router and is **deny by default**; a CI test walks every route and fails if one has no role or public marker.
- New users request access; an admin approves or rejects it in the ops UI, which adds the user to a role group. The requester is notified by email (SES).

## Cost control

A single **billing-control state machine** routes on the `action` field of its input:

- Scheduled STOP/START payloads implement nightly and weekend shutdown of ECS services (desired count → 0, back to 1 on weekday mornings).
- A daily run with no payload routes to the default branch: a **billing-guard Lambda** compares month-to-date Cost Explorer spend against a threshold. On breach it alerts via SNS, disables all of the project's scheduled EventBridge rules (except its own, so monitoring continues), and drains all ECS services to zero.
- A manual RESTORE action re-enables rules and restores services.

ECS desired counts are runtime state: Terraform ignores changes to them so a plan never fights the scheduler. AI spend is capped too: a per-user and a global daily limit on Ops Chat, checked before every turn and enforced from recorded call costs.

## AI layer

All model calls go through one layer on Bedrock (Claude Sonnet 4.6, prompt caching on). **Every call is recorded** — tokens in/out, cache tokens, latency, cost — in Aurora DSQL, attributed to a user and, for pipeline work, to a job step and attempt. Admins can audit calls and spend; users see their own usage against their limit.

**Ops Chat** answers questions about the platform using 19 **read-only** tools (pipeline runs, Glue and Step Functions state, ECS status, logs, Cost Explorer, architecture knowledge). Tools never take infrastructure names from the model, never return account IDs, and cannot change anything — actions that do are explicit admin endpoints that the chat never calls. Chat turns run in the background and the UI polls for progress, because the CDN and Amplify buffer streamed responses.

## Change pipeline

A change request (repo, title, description, acceptance criteria) becomes a pull request in five steps, with a human at both ends:

1. **Request → approval (gate 1).** A user submits; an admin approves or rejects with a note.
2. **Code context sync.** The repo is downloaded as a tarball over HTTPS and stored in S3, one object per file, keyed by commit SHA. The job records that SHA as its base — every later step uses it.
3. **Generate artifacts.** Two model calls: a *planner* picks the files to touch from the file tree; an *author* sees only those files and returns exact find-and-replace edits (or full content for new files), applied in code. The repo's own `CLAUDE.md` is passed as conventions. A path guard (no absolute paths, `..`, or `.github/`) and an output guard check everything written; repo content reaches the model only as marked data.
4. **Sandbox build.** CodeBuild overlays the generated files on the exact base snapshot and runs the repo's checks (ruff/mypy/pytest, or lint/`tsc`/build for the UI). On failure the model gets the failing logs and fixes the files, up to **three** rounds.
5. **Pull request and notification.** One atomic commit on `change/{job_id}` through the Git Data API, one PR (reused on re-run) with the build evidence in the body. If checks still fail after three fixes, the PR is raised as a draft with a "manual fix required" banner. SNS notifies on completion and on any failure.

**A person reviews and merges the PR (gate 2);** the pipeline only ever writes `change/*` branches. Steps run in-process on bounded background threads behind an HTTP 202, share one job state, and write each result in a single transaction. A step claims itself atomically and keeps a heartbeat; after a deploy or crash a silent step is taken over or marked interrupted, and running the action again resumes it. Side effects are safe to repeat: S3 artifacts overwrite, an existing PR is reused. Job and step state, and the per-step AI usage, live in Aurora DSQL.

## Data stores

- **DynamoDB** — pipeline executions and access requests, written by the Lambdas and the gateway.
- **Aurora DSQL** — AI call audit, daily spend counters, chat sessions/messages, jobs, steps and events. Chosen because audit data is queried in ways nobody can predict in advance (per-user spend, failing steps, who approved what), which is SQL work; it is serverless, costs nothing idle and fits the free tier — the single approved exception to "no RDS". Schema changes are versioned migrations applied at gateway boot.
- **S3** — the medallion layers, plus a change-pipeline workspace bucket (repo snapshots, generated files, sandbox results) with per-prefix lifecycle rules.

## Infrastructure as code

Terraform modules are layered (storage, catalog/ETL, functions, containers, eventing, notifications, email, identity, parameters, IAM), with cross-module references via remote state and a defined apply order. IAM policies are centralized in a dedicated module. Nothing is hardcoded: bucket names carry random suffixes and are published to SSM Parameter Store, from which CI and task definitions resolve them at deploy time. Application code and infrastructure deploy through separate CI pipelines mapped to environments by branch.

## Selected lessons learned

- **Step Functions Choice states need `IsPresent` guards** before evaluating any path that might be absent — otherwise the runtime throws even when a Default branch exists.
- **Fargate Spot + Route53 is a viable ALB replacement** for low-traffic services, but only with event-driven IP reconciliation and generous Lambda timeouts (IP propagation lags task RUNNING status).
- **Poll, don't stream, behind a buffering edge.** Amplify and CloudFront hold streamed responses, so long AI work runs in the background with a 202 and the UI polls.
- **Make every side effect safe to repeat.** Resume is just "run the step again", which only works if artifacts overwrite, PRs are reused and counters change in the same transaction as the row that justifies them.
- **Deny by default, and test it structurally.** A route-coverage test fails the build when any endpoint lacks a declared role — cheaper than reviewing every route by eye.
- **Treat repo content and job text as data, not instructions.** Marker-delimited prompts, a planner that never sees file contents, and path and output guards limit what an injected line can do.
