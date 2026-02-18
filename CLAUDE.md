# CLAUDE.md — Project Intelligence

This file provides context for Claude Code sessions working on this repository.

## Current State (2026-02-18)

### What's Working

The end-to-end pipeline is fully operational:
- GitHub issue with rocket reaction triggers the agent via issue-poller → agent-builder workflow
- AgentCore container starts, clones repo, runs Claude SDK against Bedrock
- Agent generates code, commits to `agent-runtime` branch, pushes via post-commit hook
- deploy-preview workflow builds the app and deploys to CloudFront
- Agent subprocess output is visible in CloudWatch via Python logging/OTEL

### What's Not Working

The agent ignores the phased execution plan (shared/ → backend/ → frontend/). On the last run (issue #14), it built a **frontend-only** React app with Dexie/IndexedDB local storage instead of the full-stack monorepo with Zod schemas, Lambda handlers, and CDK infrastructure. It got 192/192 tests passing but none of the backend or infrastructure exists. The BUILD_PLAN and system prompt need stronger guardrails to force the agent to build `shared/` and `infrastructure/` before touching frontend code.

### Key Config

| Setting | Value |
|---------|-------|
| Runtime ID | `claude_code_reinvent-1eBYMO7kHw` |
| Runtime version | 8 |
| ECR image | `669298908997.dkr.ecr.us-east-1.amazonaws.com/claude-code-reinvent:latest` |
| Execution role | `arn:aws:iam::669298908997:role/claude-code-agentcore-role` |
| Model | `us.anthropic.claude-opus-4-6-v1` |
| Region | `us-east-1` |
| GitHub repo | `KBB99/riv2025-long-horizon-coding-agent-demo` |
| Working branch | `kb/improved-harness` (harness code) |
| Agent output branch | `agent-runtime` (generated app) |

### Quick Start: Run a New Test

```bash
# 1. Reset all state (deletes agent-runtime branch, clears SSM, empties S3)
make reset

# 2. (If you changed code/prompts) Rebuild and push Docker image
make show-config  # get ECR_URI
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 669298908997.dkr.ecr.us-east-1.amazonaws.com
docker build --platform linux/arm64 -t 669298908997.dkr.ecr.us-east-1.amazonaws.com/claude-code-reinvent:latest .
docker push 669298908997.dkr.ecr.us-east-1.amazonaws.com/claude-code-reinvent:latest

# 3. (If image changed) Update runtime to pick up new image + env vars
make update-runtime-env

# 4. Wait for runtime to be READY (~15-20s)
make get-runtime  # check "status": "READY"

# 5. Create a GitHub issue and trigger the agent
gh issue create --repo KBB99/riv2025-long-horizon-coding-agent-demo \
  --title "[MVP] Canopy Build" \
  --body "Build the Canopy app as specified in BUILD_PLAN.md."
gh api repos/KBB99/riv2025-long-horizon-coding-agent-demo/issues/ISSUE_NUM/reactions -f content=rocket

# 6. Trigger immediately (instead of waiting for 5-min poller)
gh workflow run "Agent Builder" --repo KBB99/riv2025-long-horizon-coding-agent-demo -f issue_number=ISSUE_NUM

# 7. Monitor
gh run list --repo KBB99/riv2025-long-horizon-coding-agent-demo --workflow "Agent Builder" --limit 3
gh api repos/KBB99/riv2025-long-horizon-coding-agent-demo/issues/ISSUE_NUM/comments --jq '.[].body[:200]'
```

### Monitoring the Agent

```bash
# Watch CloudWatch logs (agent subprocess output visible via OTEL)
aws logs filter-log-events \
  --log-group-name "/aws/bedrock-agentcore/runtimes/claude_code_reinvent-1eBYMO7kHw" \
  --start-time $(python3 -c "import datetime; print(int((datetime.datetime.now(datetime.timezone.utc) - datetime.timedelta(minutes=5)).timestamp() * 1000))") \
  --region us-east-1 --output json | python3 -c "
import sys, json
data = json.load(sys.stdin)
for evt in data.get('events', []):
    try:
        body = json.loads(evt['message']).get('body', '')
        if body and 'agent.entrypoint' in evt['message']:
            print(body[:250])
    except: pass
"

# Check for commits on agent-runtime
gh api "repos/KBB99/riv2025-long-horizon-coding-agent-demo/commits?sha=agent-runtime&per_page=5" \
  --jq '.[] | .sha[:8] + " " + .commit.author.date + " " + (.commit.message | split("\n")[0])'

# Stop a running session
make stop-session SESSION_ID=<session-id-from-issue-comment>
```

### Lessons Learned / Pitfalls

1. **`update-runtime-env` replaces ALL env vars** — if you add a new env var to `launch` but forget to add it to `update-runtime-env` in the Makefile, it gets wiped on the next update. This was the root cause of issues #5-#13 (missing `CLAUDE_CODE_USE_BEDROCK`).
2. **`print()` is invisible in CloudWatch** — only Python `logging` module output gets captured by OTEL auto-instrumentation. Use `logger.info()` for anything you need to see.
3. **Docker images must be ARM64** — AgentCore runs on Graviton. Always build with `--platform linux/arm64`.
4. **`bypassPermissions` requires non-root** — the Claude CLI refuses `--dangerously-skip-permissions` when running as root. The Dockerfile creates a non-root `agent` user (UID 1000).
5. **The agent takes 30-40 min before its first commit** — it builds up a large batch of files before committing. The background push loop and post-commit hook handle pushing once commits exist.

## Architecture

The system is an autonomous coding agent that builds full-stack applications from GitHub issues.

### Execution Flow

```
GitHub Issue (with reaction)
  → issue-poller.yml (detects approved issues)
  → agent-builder.yml (acquires lock, invokes AgentCore)
  → bedrock_entrypoint.py (clones repo, resolves config)
  → claude_code.py (wraps Claude SDK, runs agent session)
  → Claude builds the app, commits to agent-runtime branch
  → deploy-preview.yml (builds and deploys to CloudFront)
```

### Key Files

| File | Purpose |
|------|---------|
| `bedrock_entrypoint.py` | Main orchestrator. Clones the repo, reads `PROJECT_NAME`, resolves the build plan path, sets up environment, and spawns `claude_code.py` as a subprocess. Handles both GitHub mode (from issues) and legacy mode. |
| `claude_code.py` | Agent session manager. Wraps the Claude Agent SDK. Loads `BUILD_PLAN.md`, system prompts, and example tests. Manages the conversation loop, screenshots, and git operations. |
| `src/github_integration.py` | GitHub API wrapper. Posts comments, manages labels, uploads screenshots, creates/updates issues. |
| `src/git_operations.py` | Git commit and push logic. Handles periodic commits to `agent-runtime` branch. |
| `src/cloudwatch_metrics.py` | Heartbeat metrics for health monitoring. |
| `Makefile` | All management commands: `launch`, `reset`, `deploy-infra`, `show-config`, etc. |
| `infrastructure/lib/claude-code-stack.ts` | CDK stack defining ECR, S3 buckets, CloudFront distributions, IAM roles. |

### GitHub Actions Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `issue-poller.yml` | Cron (every 5 min) | Finds approved issues, triggers agent-builder |
| `agent-builder.yml` | `workflow_dispatch` from poller | Invokes Bedrock AgentCore session |
| `deploy-preview.yml` | Push to `agent-runtime` | Builds the generated app and deploys to CloudFront |

## How PROJECT_NAME Works

`PROJECT_NAME` is set in the Makefile (default: `canopy`) and passed as an environment variable through AgentCore to the container.

1. `bedrock_entrypoint.py` reads `os.environ.get("PROJECT_NAME", "canopy")`
2. Passes `--project {project_name}` to `claude_code.py`
3. `claude_code.py` loads files from `prompts/{project_name}/`:
   - `BUILD_PLAN.md` — the full project specification (required)
   - `EXAMPLE_TEST.txt` — example test patterns (optional)
   - `DEBUGGING_GUIDE.md` — project-specific debugging tips (optional)
   - `system_prompt.txt` — project-specific system prompt (optional)
4. Shared prompts are always loaded from `prompts/`:
   - `system_prompt.txt` — base system prompt
   - `DEBUGGING_GUIDE.md` — general debugging guide
   - `FRONTEND_AESTHETICS_GUIDE.md` — UI design guidance

### Creating a New Project

1. Create `prompts/{project-name}/BUILD_PLAN.md` with full specification
2. Optionally add `EXAMPLE_TEST.txt`, `DEBUGGING_GUIDE.md`, `system_prompt.txt`
3. Rebuild and push the Docker image (prompts are baked into the image — see "Deploying Changes")
4. Launch: `make launch PROJECT_NAME={project-name}`

## Deploying Changes

Changes to agent code (`claude_code.py`, `bedrock_entrypoint.py`, `src/`), prompts (`prompts/`), or the scaffold template require rebuilding and pushing the Docker image. The Dockerfile (`COPY prompts/ ./prompts/`) bakes prompts into the image at `/app/prompts/` — merging to `main` alone does NOT update the running agent.

### Full deployment sequence

```bash
# 1. Deploy CDK infrastructure (if stack resources changed)
make deploy-infra

# 2. Build and push the Docker image to ECR
#    Get the ECR URI from stack outputs:
make show-config   # look for ECR_URI

#    Login, build, push:
aws ecr get-login-password --region us-east-1 --profile default | \
  docker login --username AWS --password-stdin <ECR_URI>
docker build --platform linux/arm64 -t <ECR_URI>:latest .
docker push <ECR_URI>:latest

# 3. (Optional) Update runtime env vars if they changed
make update-runtime-env

# 4. Reset agent state for a fresh run
make reset

# 5. Launch a new session (or let issue-poller trigger one)
make launch
```

### What requires an image rebuild

| Change | Rebuild image? | CDK deploy? |
|--------|---------------|-------------|
| `prompts/` (BUILD_PLAN, system_prompt) | **Yes** | No |
| `claude_code.py`, `bedrock_entrypoint.py` | **Yes** | No |
| `src/*.py` (git_operations, github_integration) | **Yes** | No |
| `frontend-scaffold-template/` | **Yes** | No |
| `infrastructure/lib/claude-code-stack.ts` | No | **Yes** |
| `Makefile` (env var defaults) | No | No (use `make update-runtime-env`) |
| `.github/workflows/` | No | No (picked up from `main` by GitHub Actions) |

## Resetting Agent State

Run `make reset` to wipe everything and start fresh. This:

- Deletes `agent-runtime` branch (local + remote)
- Closes all open issues with `agent-building` label
- Deletes SSM parameters: `/claude-code/current-issue`, `/claude-code/session-id`, `/claude-code/infra/deploy-state`
- Empties S3 screenshots and previews buckets
- Invalidates CloudFront caches

## CDK Context Variables

Pass via `--context key=value` when deploying infrastructure:

| Variable | Default | Description |
|----------|---------|-------------|
| `projectName` | `claude-code` | Prefix for all AWS resource names |
| `environment` | `reinvent` | Environment suffix (affects secret paths, stack name) |
| `vpcId` | (creates new) | Use existing VPC instead of creating one |
| `agentCoreRoleName` | (none) | Existing IAM role name for AgentCore |
| `agentRuntimeId` | `YOUR_AGENT_RUNTIME_ID` | AgentCore runtime ID (for log group config) |
| `githubActionsUserName` | (none) | Existing IAM user for GitHub Actions |
| `logRetentionDays` | `7` | CloudWatch log retention period |

## Common Issues and Fixes

### Labels missing
Workflows fail silently if `agent-building`, `agent-complete`, or `tests-failed` labels don't exist. Create them:
```bash
gh api repos/OWNER/REPO/labels -f name="agent-building" -f color="FBCA04"
gh api repos/OWNER/REPO/labels -f name="agent-complete" -f color="0E8A16"
gh api repos/OWNER/REPO/labels -f name="tests-failed" -f color="D93F0B"
```

### deploy-preview can't find dist/
The deploy-preview workflow searches for `dist/` in workspace subdirectories. If the generated app uses a different build output directory, update the find pattern in `.github/workflows/deploy-preview.yml`.

### Agent stuck or stale session
If the agent appears stuck, check the SSM parameter `/claude-code/session-id` for the current session. Use `make stop-session SESSION_ID=xxx` to stop it, or `make reset` to clear everything.

### CloudFront returns old content
Run `make reset` which includes CloudFront cache invalidation, or manually invalidate:
```bash
aws cloudfront create-invalidation --distribution-id DIST_ID --paths "/*"
```

### f-string escaping in SSM instructions
When modifying `bedrock_entrypoint.py`, JSON braces inside f-strings must be double-escaped (`{{` and `}}`). This has been a source of bugs in SSM parameter instructions.
