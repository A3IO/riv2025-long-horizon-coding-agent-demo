# CLAUDE.md — Project Intelligence

This file provides context for Claude Code sessions working on this repository.

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
