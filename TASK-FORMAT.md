# Task Format Guide

## Compact Task Format

Tasks reference repos by ID. The executor auto-resolves full context (paths, dependencies, APIs, tech stack) from the repo registry.

```json
{
  "batch": "fix-token-refresh",
  "description": "Fix JWT refresh failing after 24h in production",
  "author": "ayush",
  "priority": 1,
  "model": "sonnet",

  "repos": ["backend", "product-api-common"],
  "goal": "Fix the token refresh endpoint that fails after 24h due to stale refresh token storage",

  "crossRepo": {
    "sharedModels": ["UserToken", "RefreshToken"],
    "sharedAPIs": ["/api/v1/auth/refresh"],
    "migrationRequired": false,
    "breakingChanges": false,
    "affectedEnvironments": ["staging", "production"]
  },

  "tasks": [
    {
      "title": "Diagnose the refresh failure",
      "prompt": "Read the auth refresh handler. Check the token expiry logic. Look at recent error logs for 'refresh' failures."
    },
    {
      "title": "Fix the refresh logic",
      "prompt": "Fix the identified issue. Ensure refresh tokens are rotated correctly and old ones invalidated.",
      "verification": "Run: npm test -- --grep refresh",
      "expectedResult": "All refresh tests pass"
    },
    {
      "title": "Create PR",
      "prompt": "Create a branch fix/token-refresh, commit, push, and open a PR with a clear description of what was fixed and why.",
      "verification": "Run: gh pr list --state open --json title",
      "expectedResult": "PR exists with token refresh fix"
    }
  ]
}
```

## Field Reference

### Required Fields

| Field | Description |
|-------|-------------|
| `batch` | Unique identifier for this batch (used as session name) |
| `description` | One-line description of what this batch does |
| `author` | GitHub username of the person who created the batch |
| `tasks` | Array of tasks to execute sequentially |
| `tasks[].title` | Short title for the task |
| `tasks[].prompt` | Instructions for Claude (keep focused, context is auto-injected) |

### Repo Context Fields

| Field | Description |
|-------|-------------|
| `repos` | Array of repo IDs (e.g., `["backend", "caishen"]`). Auto-resolves paths, dependencies, tech stack |
| `goal` | What you're trying to achieve across these repos (injected as context) |
| `crossRepo.sharedModels` | Data models/schemas touched across repos |
| `crossRepo.sharedAPIs` | API endpoints involved in cross-repo communication |
| `crossRepo.migrationRequired` | `true` if a DB migration is needed |
| `crossRepo.breakingChanges` | `true` if this introduces breaking API changes |
| `crossRepo.affectedEnvironments` | Which environments are affected (`["staging", "production"]`) |

### Optional Fields

| Field | Default | Description |
|-------|---------|-------------|
| `priority` | 50 | 1 (highest) to 99 (lowest) |
| `model` | haiku | Claude model: `haiku`, `sonnet`, `opus` |
| `project` | — | Override working directory for all tasks |
| `tasks[].verification` | — | Command to verify task completion |
| `tasks[].expectedResult` | — | What success looks like |

## Available Repo IDs

### Critical (core business logic)
- `backend` — Core API (accounts, portfolios, auth)
- `caishen` — Investment engine (portfolios, rebalancing)
- `alfred` — Workflow orchestration
- `FMS` — Financial management (reporting, reconciliation)
- `proximus` — API gateway
- `product-core-db` — Shared database schemas

### High Priority
- `product-api-common` — Shared utilities across services
- `product-fastify-plugins` — Shared Fastify plugins
- `mark8` — Market data service
- `www2` — Web frontend (React/Next.js)
- `product-mobile-reactnative` — Mobile app
- `svava-router` — Request routing

### Medium
- `product-admin-api` — Internal admin API
- `product-admin-web` — Internal admin UI
- `chatbot-app` — Customer chatbot
- `product-onboarding-ui` — Onboarding flow
- `data-swanalytics` — Data analytics
- `nouri`, `apollo`, `atlas`, `akari`, `westworld` — Supporting services

### Infrastructure & Tooling
- `syfe-helm-values` — Helm chart values
- `alfred-manifests` — K8s manifests for Alfred
- `dev-github-workflows` — Shared GitHub Actions
- `syfe-monitoring` — Monitoring config
- `syfeCli` — Internal CLI
- `syfe-ai-skills` — Claude Code skills
- `syfe-mcp-proxy` — MCP proxy

### Testing
- `api-tests` — E2E API tests
- `stf` — Syfe Test Framework

## What Gets Auto-Injected

When you specify `"repos": ["backend", "caishen"]`, the executor automatically prepends to the first task:

```markdown
# Repository Context (auto-injected)

## backend
- Path: ~/Documents/backend
- Description: Core Syfe backend API
- Tech: Node.js, Fastify, PostgreSQL
- Depends on: product-core-db, product-api-common, product-fastify-plugins
- Used by: alfred, caishen, FMS, proximus, www2, product-mobile-reactnative
- Exposes: REST /api/v1/*, internal gRPC
- ⚠ Note: Central service — changes here ripple across many consumers

## caishen
- Path: ~/Documents/caishen
- Description: Investment management, portfolio calculations
- Tech: Node.js, Fastify
- Depends on: backend, product-core-db, product-api-common
- Used by: FMS, alfred

## Potentially Affected Repos
- alfred (Orchestration service)
- FMS (Financial Management System)
- proximus (API gateway)
- www2 (Web frontend)

## Cross-Repo Communication
- caishen consumes backend — check interface compatibility
```

Claude also gets `--add-dir` access to all referenced repo paths.

## Tips for Writing Good Tasks

1. **Keep prompts focused** — repo context is auto-injected, don't repeat it
2. **Use `goal`** — tells Claude the big picture across repos
3. **Mark `crossRepo.breakingChanges: true`** — Claude will check dependent repos
4. **One batch per logical change** — "Fix auth bug" not "Fix auth + add feature"
5. **Verification is optional but valuable** — helps catch failures early
6. **Use `priority: 1`** for urgent fixes, default `50` for regular work
