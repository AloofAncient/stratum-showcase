# CLAUDE.md — stratum-showcase

## What this repo is

Public showcase for the Stratum platform: `README.md`, `docs/ARCHITECTURE.md` (the one maintained architecture doc), the interactive `stratum-architecture.html` map and `assets/` screenshots. **Docs only** — no application code, no Terraform. Personal/educational project; **cost is always the primary constraint**. Region `us-east-1`, active environment DEV only.

The source repos are private. This repo describes the system for outside readers; it never deploys anything.

## Collaboration conventions — always follow these

- **Edit only inside this repo.** Never change files outside `stratum-showcase`, with one exception: append your own entry to `E:\Stratum\docs\v2\PROGRESS.md` (newest first, template in that file, ≤ 15 lines, today's real date).
- **Other repos are read-only.** Read any repo to get facts; never edit, commit or run anything that changes them. This applies every session — do not drift.
- **Keep responses short and precise.** Answer in as few words as possible: no preamble, no recap of what was just done, no long option lists. Add detail only when asked.
- **Discuss before implementing** anything beyond a doc edit that was asked for. **No new files without discussion.**
- **Match existing file patterns exactly** (structure, headings, tone).
- **Verify before claiming** — check the source repos or official docs; say so when unsure.
- **Git:** commit/push only when Vivek asks. Never commit directly to `main` without asking. Never commit secrets, `.env`, or `.mcp.json`.
- Do not touch the untracked `job-*` folders — unrelated to Stratum.

## Content rules (public docs)

- **No references to the retired MCP server or `stratum-aws-ai-agents`** — not in text, diagrams, tech stack or screenshots. Drop every mention.
- **Describe the current system only** (gateway v2, Bedrock, Ops Chat, change pipeline), not the v1 history.
- No account IDs, ARNs, bucket names, internal URLs beyond the public demo links, or security-config specifics.
- Source of truth for what is true: `docs/v2/PLAN.md` and the repo `CLAUDE.md` files. If they disagree with this repo, the plan wins.
- **Screenshots:** if a section needs a new or replaced screenshot, point it out to Vivek (what to capture, which file name in `assets/`) — do not leave stale images in place.
- Keep `README.md`, `docs/ARCHITECTURE.md` and `stratum-architecture.html` consistent with each other.

---

## Related repos (read-only from here)

| Repo | Purpose |
|------|---------|
| `stratum-aws-infra` | All AWS infrastructure — Terraform, HCP Terraform |
| `stratum-aws-app` | Glue ETL scripts + Lambda function code |
| `stratum-gateway-aws` | FastAPI platform API on ECS Fargate Spot — data API, Ops Chat, change pipeline |
| `stratum-aws-ops-ui` | Next.js ops UI on Amplify — Cognito Google SSO |

## Branch → environment mapping (as described in the docs)

`develop` → DEV · `release` → UAT · `master` → PROD (only DEV is deployed).
