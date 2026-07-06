# STRATUM Architecture (Public)

This is the public architecture overview. Operational runbook details, security configuration specifics, and internal invariants are kept in the private repo.

## System overview

STRATUM collects financial market data weekly, processes it through a medallion pipeline (raw → bronze → silver → gold), and serves aggregated results through a REST API. The primary design constraint is **cost**: no fixed-hourly resources anywhere (no ALB, no NAT Gateway, no RDS), compute on Fargate Spot, and automated shutdown outside working hours. Deployed in us-east-1, entirely via Terraform (HCP Terraform), with branch → environment promotion: `develop → DEV`, `release → UAT`, `master → PROD`.

## Data pipeline

An EventBridge rule fires weekly and starts a Step Functions state machine that:

1. Records the run in DynamoDB (status RUNNING) and sends an SNS start notification (best-effort — a missed email is not a reason to skip the run).
2. Invokes the **data-ingestor Lambda**, which fetches market data (US tech, Indian NSE equities, crypto, indices, commodities) from the Yahoo Finance chart API via direct HTTP and writes one JSON file per symbol to the S3 raw bucket under a date-partitioned prefix.
3. Runs three Glue jobs sequentially using `.sync` integration: **raw→bronze** (schema enforcement, dedup), **bronze→silver** (business transforms, enrichment), **silver→gold** (aggregations and KPIs: weekly summary, top/worst performers, volatility report, asset-class summary, per-symbol detail). All output is Parquet.
4. Marks the run SUCCEEDED/FAILED in DynamoDB and sends a completion/failure notification.

The **gold layer is the serving layer** — the API reads it directly from S3, resolving the latest `ingested_date` partition at query time. There is no database between the pipeline and the API.

## Serving layer

A Flask API runs on ECS Fargate Spot in public subnets. There is no load balancer: CloudFront points at a Route53 A record holding the current task's public IP. When Spot replaces the task, an EventBridge rule on ECS Task State Change events triggers a Lambda that reads the new ENI's public IP and upserts the A record. The Lambda retries internally because the IP isn't always assigned the instant ECS reports RUNNING.

Two auth paths:

- **Browser users:** Google SSO via Cognito Hosted UI. A pre-token-generation Lambda enforces an approval gate — only users in an approved group receive tokens. Session is an HTTP-only cookie.
- **Programmatic:** API-key/JWT Bearer auth; credentials verified once at token issuance, then stateless JWT verification per request.

`openapi.yaml` is the source of truth for all documented endpoints, rendered via Swagger UI.

## Cost control

A single **billing-control state machine** routes on the `action` field of its input:

- Scheduled STOP/START payloads implement nightly and weekend shutdown of ECS services (desired count → 0, back to 1 on weekday mornings).
- A daily run with no payload routes to the default branch: a **billing-guard Lambda** compares month-to-date Cost Explorer spend against a threshold. On breach it alerts via SNS, disables all of the project's scheduled EventBridge rules (except its own, so monitoring continues), and drains all ECS services to zero.
- A manual RESTORE action re-enables rules and restores services.

ECS desired counts are runtime state: Terraform ignores changes to them so a plan never fights the scheduler.

## AI operations layer

Two services make the platform AI-operable:

**Multi-agent assistant** (ECS Fargate Spot, 0.25 vCPU): an orchestrator LLM (Groq Llama 3.3 70B, falling back to Llama 3.1 8B on rate limits) routes user prompts to five specialists — billing, pipeline, infrastructure, architecture knowledge, and GitHub. Conversation history persists in DynamoDB keyed by session, enabling multi-turn conversations across HTTP requests. Sensitive actions (service restore, budget changes, PR creation/review requests) are guarded: they write a pending-approval record with a 1-hour TTL and execute only after explicit human approval via an approvals endpoint. Scheduled EventBridge rules can drive proactive health-check and post-pipeline-summary conversations.

**MCP server** (ECS Fargate Spot): exposes 15 AWS observability tools to MCP-compatible AI clients over streamable HTTP — S3 inspection, CloudWatch log search and error fingerprinting, pipeline run history, Step Functions execution detail, Glue job runs, EventBridge rule status, ECS service/task status, Cost Explorer spend and per-service breakdown, Route53 record lookup, and Cognito user status — plus an architecture resource so the client can load full system context before answering. It runs under a dedicated IAM task role, deliberately separate from the API's role, so broad read-only discovery permissions don't bleed into the serving path.

## Infrastructure as code

Terraform modules are layered (storage, catalog/ETL, functions, containers, eventing, notifications, parameters, IAM), with cross-module references via remote state and a defined apply order. IAM policies are centralized in a dedicated module. Nothing is hardcoded: bucket names carry random suffixes and are published to SSM Parameter Store, from which CI and task definitions resolve them at deploy time. Application code and infrastructure deploy through separate CI pipelines mapped to environments by branch.

## Selected lessons learned

- **Step Functions Choice states need `IsPresent` guards** before evaluating any path that might be absent — otherwise the runtime throws even when a Default branch exists.
- **Fargate Spot + Route53 is a viable ALB replacement** for low-traffic services, but only with event-driven IP reconciliation and generous Lambda timeouts (IP propagation lags task RUNNING status).
- **Model fallback chains matter** for free-tier LLM APIs: every agent needs a cheaper fallback on rate-limit errors or the whole assistant becomes flaky.
- **Cookie forwarding is the silent SSO killer** behind CloudFront: the default strips all cookies, and nothing errors — the app just never sees the session.
