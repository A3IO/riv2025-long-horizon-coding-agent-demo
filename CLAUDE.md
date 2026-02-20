# CLAUDE.md — Project Intelligence

This file provides context for Claude Code sessions working on this repository.

## Current State (2026-02-20)

### What's Working

The end-to-end pipeline is fully operational:
- GitHub issue with rocket reaction triggers the agent via issue-poller → agent-builder workflow
- AgentCore container starts, clones repo, runs Claude SDK against Bedrock
- Agent generates code, commits to `agent-runtime` branch, pushes via post-commit hook
- deploy-preview workflow builds the app and deploys to CloudFront
- deploy-infrastructure workflow builds shared/backend, runs CDK tests, and deploys the stack
- Agent subprocess output is visible in CloudWatch via Python logging/OTEL

### Issue #22 — Backend Test Verification Fix (SUCCESS)

**The problem (issue #17):** Despite building the backend correctly (shared → infra → backend → frontend), all 220 tests were frontend-only. Zero shared/infrastructure/backend tests.

**Root cause:** A hardcoded screenshot gate in `src/security.py` prevented marking ANY test as passing without a Playwright screenshot + console file. Since there was no verification path for backend tests, the agent rationally wrote only frontend tests.

**Fix applied (commits on `kb/improved-harness`):**
- Created `frontend-scaffold-template/backend-verify.cjs` — runs shell commands and produces `-result.txt` + `-console.txt` artifacts
- Added alternative verification path in `src/security.py` — accepts `-result.txt` with `VERIFIED_BY: backend-verify.cjs` sentinel instead of requiring screenshots
- Added AWS data-plane commands to security allowlist (DynamoDB scan/query, CloudWatch logs, Lambda invoke)
- Added IAM permissions in CDK stack (`AgentCoreBackendTestPolicy`) for canopy-* scoped resources
- Updated prompts with backend-verify.cjs examples for all categories
- Updated `claude_code.py` initial + continuation messages with category-specific verification guidance

**Result (issue #22):** 54/54 tests passing — 3 shared, 7 infrastructure, 6 backend, 38 frontend. 18 tests use backend-verify.cjs, 36 use playwright variants. Build order correct: shared → infra → backend → frontend.

### Remaining Issues

#### Frontend not connected to deployed backend
The agent builds a working frontend with a good API client architecture (tryApi with localStorage fallback), but `VITE_API_URL` is never set. The frontend defaults to localStorage for all data. The agent never checks SSM deploy-state for the API URL and never creates `frontend/.env`.

#### CI/CD deploy-infrastructure failed 3 times
The deploy-infrastructure workflow triggered on 3 commits but failed every time at `npm test` with "Cannot find asset at generated-app/backend/dist". The workflow was missing build steps for shared/ and backend/. **Fixed in commit `4a8867c`** — added shared + backend build steps before CDK tests.

#### Agent unaware of CI/CD pipeline
CloudWatch log search for "deploy", "workflow", "CI", "deploy-state", "ssm", "apiUrl", "VITE_API" returned zero events. The agent has no feedback loop from GitHub Actions and never checks whether its infrastructure was deployed.

#### Next: Agent should use deployed CDK resources
The agent writes CDK infrastructure and backend Lambda handlers in the same phase, but the backend code isn't actually deployed or connected. The prompts need to instruct the agent to:
1. Commit CDK infrastructure separately and wait for CI/CD to deploy it
2. Check SSM deploy-state to confirm deployment succeeded and get the API URL
3. Write backend handlers that reference the deployed resources
4. Wire `VITE_API_URL` into the frontend so it uses the real API

### Previous Fixes (for reference)

**Phase ordering (issue #17, commit `8dfdae7`):** Replaced grading language with phase-completion incentives. Agent went from building frontend-only to building shared → infra → backend → frontend correctly.

**IAM role mismatch (issue #20-21):** CDK deployed policies to wrong IAM role (`AmazonBedrockAgentCoreSDKRuntime-*` instead of `claude-code-agentcore-role`). Container silently crashed. Fixed by adding `AGENTCORE_ROLE_NAME` and `VPC_ID` as Makefile defaults for `deploy-infra`.

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
5. **The agent takes 20-30 min before its first commit** — it builds up a large batch of files before committing. The background push loop and post-commit hook handle pushing once commits exist.
6. **Incentives shape agent behavior more than instructions** — saying "graded on UI quality" and "200 tests" caused the agent to skip backend/infra entirely and build a frontend-only app. Replacing grading language with phase-completion scoring fixed the build order immediately (issue #17 vs #14).
7. **Test count suggestions are weak constraints** — asking for "~50 tests" still produced 210+. If strict test count control is needed, it may require enforcement in the harness code rather than the prompt.
8. **CDK `agentCoreRoleName` must match the container's execution role** — The container runs as `claude-code-agentcore-role`, NOT the `AmazonBedrockAgentCoreSDKRuntime-*` role. If you pass the wrong role name to `cdk deploy -c agentCoreRoleName=...`, the Secrets Manager / SSM / CloudWatch policies attach to the wrong role and the container silently crashes (no logs after handler init). The Makefile's `deploy-infra` target now passes the correct defaults automatically.
9. **CDK deploy without `vpcId` creates a new VPC** — The account has a VPC limit. Always pass `-c vpcId=vpc-04be60df8488bb6e5` (or use `make deploy-infra` which does this automatically).

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
