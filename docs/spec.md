# DigitalOcean Deployment Specification

Status: Draft v1
Target system: Symphony (Elixir reference implementation)
Platform: DigitalOcean App Platform (managed apps)

## 1. Goal

Define a production deployment plan for Symphony on DigitalOcean App Platform using managed app
services, with continuous deployment triggered from the `main` branch.

## 2. Scope

This specification covers:

- Infrastructure and account prerequisites.
- App Platform service configuration for the Elixir runtime.
- Secret and environment variable requirements.
- Continuous deployment policy with `main` as the deployment branch.
- Post-deployment validation and rollback checks.

This specification does not cover:

- Rewriting Symphony architecture.
- Multi-region active-active deployments.
- Non-DigitalOcean hosting targets.

## 3. Deployment Architecture

Deploy one managed App Platform service from this repository:

- Service type: `Worker` (always-on process).
- Source directory: `elixir/`.
- Build strategy: buildpack (auto-detected for Elixir) or Dockerfile fallback.
- Runtime command: run Symphony escript with explicit guardrails acknowledgement.

Recommended topology:

- Worker process:
  - Runs polling/orchestration loop continuously.
  - Uses persistent external services for tracker and model APIs.
- Optional second component (`Web Service`) if public dashboard/API access is required:
  - Runs same codebase with `--port "$PORT"` enabled.
  - Exposes dashboard/API endpoints for operational checks.

## 4. Preconditions and Prerequisites

All items below are mandatory before first production deployment.

### 4.1 Accounts and Access

- DigitalOcean account with billing enabled.
- DigitalOcean project created for Symphony.
- GitHub repository admin access for app installation and webhook setup.
- Linear workspace token creation rights.
- OpenAI API key provisioning for Codex usage.

### 4.2 Repository and Branching

- Repository contains valid `elixir/WORKFLOW.md` for the target environment.
- `main` branch exists and is the only deployment branch.
- Branch protection on `main` requires CI and review before merge.
- No direct pushes to `main` except through approved PR flow.

### 4.3 Runtime Toolchain and Build Inputs

- Elixir/Erlang versions pinned (see `elixir/mise.toml` and `elixir/mix.exs`).
- Dependencies resolvable from `mix.lock`.
- Build command is able to produce `elixir/bin/symphony` (via `mix build`).

### 4.4 Required Secrets and Environment Variables

Do not store real values in git or logs.

Required in App Platform environment settings:

- `LINEAR_API_KEY` (secret): Linear API token.
- `OPENAI_API_KEY` (secret): OpenAI key for Codex app-server calls.
- `WORKFLOW_PATH` (plain): absolute path to workflow file inside container, default
  `/workspace/elixir/WORKFLOW.md`.
- `SYMPHONY_LOGS_ROOT` (plain): writable logs path, default `/workspace/log`.

Optional but recommended:

- `LINEAR_ASSIGNEE` (secret/plain): optional assignee filter.
- `PORT` (App Platform managed): only required when deploying an optional web component.

### 4.5 Workflow Contract Validation

Before deployment, validate that `elixir/WORKFLOW.md` includes:

- `tracker.kind: linear`.
- `tracker.project_slug` value for the target Linear project.
- Active/terminal states matching team workflow.
- `codex.command` set for the intended model/runtime.

### 4.6 Operational Readiness

- Alerting destination configured (Slack/email/PagerDuty).
- Log retention window defined.
- On-call owner defined for deployment failures.
- Rollback procedure rehearsed at least once in staging.

## 5. DigitalOcean App Platform Specification

### 5.1 App Resource Configuration

- App type: Managed App Platform app.
- Component name: `symphony-worker`.
- Component type: Worker.
- GitHub source:
  - Repo: this repository.
  - Branch: `main`.
  - Auto-deploy: enabled.
- Source directory: `elixir`.

### 5.2 Build and Run Commands

Use commands equivalent to:

- Build command:
  - `mix deps.get --only prod && MIX_ENV=prod mix build`
- Run command:
  - `./bin/symphony --i-understand-that-this-will-be-running-without-the-usual-guardrails --logs-root "$SYMPHONY_LOGS_ROOT" "$WORKFLOW_PATH"`

Optional web component run command:

- `./bin/symphony --i-understand-that-this-will-be-running-without-the-usual-guardrails --logs-root "$SYMPHONY_LOGS_ROOT" --port "$PORT" "$WORKFLOW_PATH"`

### 5.3 Instance Sizing

Start baseline:

- 1 worker instance.
- Size tier: Basic with sufficient memory for Elixir VM + Codex subprocesses.

Scale guidance:

- Increase instance size before horizontal scaling when memory pressure is observed.
- Keep max concurrent agent settings in `WORKFLOW.md` aligned with instance capacity.

### 5.4 Health, Restart, and Failure Policy

- Auto-restart on process crash enabled (platform default).
- Deployment failure must block promotion.
- If optional web endpoint is enabled, health probe should target `/api/v1/state`.

## 6. Continuous Deployment Policy

### 6.1 Deployment Branch Contract

- `main` is the deployment branch.
- Every merge to `main` triggers a new App Platform deployment automatically.
- No manual cherry-picks to production branch.

### 6.2 CI Gate Before Deploy

A commit may reach `main` only after:

- CI workflow success (at minimum `.github/workflows/make-all.yml`).
- Required review approvals.
- No unresolved blocking review threads.

### 6.3 CD Trigger and Promotion

- Trigger source: GitHub push event on `main`.
- Build and deploy happen in App Platform managed pipeline.
- Failed deploy keeps last healthy revision serving.

### 6.4 Rollback Strategy

- Primary rollback: redeploy previous successful App Platform revision.
- Secondary rollback: revert offending commit in GitHub, merge revert to `main`, allow auto-deploy.

## 7. Post-Deployment Validation Checklist

Run after every production deployment.

### 7.1 Platform-Level Checks

- App deployment status is `Healthy` in DigitalOcean.
- Latest deployment build logs show successful `mix build` and startup.
- Restart count remains stable (no crash loop).

### 7.2 Process and Connectivity Checks

- Symphony process is running and polling interval is active.
- Linear connectivity succeeds (no auth/project slug errors).
- Codex command starts successfully from runtime environment.

### 7.3 Functional Checks

- Create or select a non-critical test issue in configured Linear project.
- Verify Symphony claims the issue and starts a workspace run.
- Verify workpad comment is created/updated by agent workflow.
- Verify state transitions progress according to workflow (`Todo` -> `In Progress` -> review state).

### 7.4 Observability Checks (Optional Web Component)

If `--port` is enabled:

- `GET /api/v1/state` returns `200` and valid JSON.
- `GET /` dashboard loads successfully.
- Issue snapshot endpoint (`/api/v1/<identifier>`) returns expected payload for active issue.

### 7.5 Security and Secret Hygiene Checks

- No secrets are printed in application logs.
- Runtime env variables marked as secrets in DigitalOcean are not exposed in UI output.
- Workflow file path and logs root are confined to expected directories.

### 7.6 Acceptance Gate

A deployment is accepted only when all checks in sections 7.1-7.5 pass.

## 8. Operational Runbook (Failure Handling)

### 8.1 Failed Build

- Inspect App Platform build logs.
- Confirm dependency fetch and `mix build` success.
- Fix in branch, merge to `main`, allow redeploy.

### 8.2 Failed Startup

- Inspect runtime logs for missing env vars or workflow parse errors.
- Validate `WORKFLOW_PATH` points to existing file.
- Validate required secrets (`LINEAR_API_KEY`, `OPENAI_API_KEY`) are present.

### 8.3 Runtime Degradation

- Check API rate limits and token exhaustion.
- Reduce `agent.max_concurrent_agents` if resource starvation occurs.
- Roll back to last healthy revision if processing halts.

## 9. Change Management

Any change to deployment branch policy, runtime command, required secrets, or post-deploy checks
must update this spec in the same PR as the implementation change.
