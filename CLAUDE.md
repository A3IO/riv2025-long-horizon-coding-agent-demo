# CLAUDE.md — Project Intelligence

This file provides context for Claude Code sessions working on this repository.

## Current State (2026-02-19)

### What's Working

The end-to-end pipeline is fully operational:
- GitHub issue with rocket reaction triggers the agent via issue-poller → agent-builder workflow
- AgentCore container starts, clones repo, runs Claude SDK against Bedrock
- Agent generates code, commits to `agent-runtime` branch, pushes via post-commit hook
- deploy-preview workflow builds the app and deploys to CloudFront
- Agent subprocess output is visible in CloudWatch via Python logging/OTEL

### Phase Ordering Fix (issue #17)

The agent previously ignored the phased execution plan (shared/ → backend/ → frontend/). On issue #14, it built a **frontend-only** React app with Dexie/IndexedDB local storage instead of the full-stack monorepo.

**Root cause:** The system prompt incentivized UI quality ("graded on quality of GUI", "pixel-perfect") and demanded 200+ tests upfront, causing the agent to rationally build a testable frontend-only app.

**Fix applied (commit `8dfdae7`):**
- Replaced grading language with phase-completion incentives ("working shared/ + infrastructure/ + backend/ scores higher than polished frontend with no backend")
- Reduced test count from 200 to ~50, weighted toward backend/infra categories (shared, infrastructure, backend, then frontend)
- Added Phased Execution section to canopy-specific system prompt (was only in top-level)
- Added `@canopy/shared` import requirements to canopy Backend Development section
- Replaced "all tests must pass" completion gates with phase gates (shared compiles, infra synths, backend builds, frontend builds)
- Simplified test verification (removed "SYSTEM WILL BLOCK YOUR EDIT" enforcement language)
- Softened continuation message (removed one-line-at-a-time test enforcement, reduced verification paranoia)

**Result (issue #17):** Agent's first commit was `feat: add shared API contract, CDK infrastructure, and Lambda handlers` — it built shared/ + infra + backend before touching frontend. Two sessions, 10 commits, live preview deployed.

### Next Investigation: Why are tests still 100% frontend?

**The problem:** Despite the prompt asking for ~50 tests with the first ~20 covering shared/infra/backend, the agent wrote 220 tests that are ALL frontend UI tests. Zero `shared`, `infrastructure`, or `backend` category tests. The phase ordering fix worked (it built the backend first), but the test composition is still entirely UI-biased.

**Hypothesis:** The bias may not come from the system prompt alone. There may be something else in the pipeline that prepopulates or biases toward frontend tests. Investigate these in order:

1. **`frontend-scaffold-template/`** — This template is copied into `generated-app/` before the agent starts. Check if it includes a `tests.json` or any test scaffolding that's frontend-only. If the agent sees existing frontend test patterns, it will follow them.
   - Look at: `frontend-scaffold-template/` directory contents (already in the repo)
   - Key question: Does the template include any `tests.json`, `playwright-test.cjs`, or example test files?

2. **`prompts/canopy/EXAMPLE_TEST.txt`** — This file is loaded by `load_example_test()` in `claude_code.py:70-95` and injected into the continuation message. If it only shows frontend test examples, the agent learns that pattern.
   - Look at: `prompts/canopy/EXAMPLE_TEST.txt` (if it exists)
   - Also check: `prompts/EXAMPLE_TEST.txt` (top-level fallback)

3. **`prompt_template.txt`** — Loaded in `create_thyme_style_message()` at `claude_code.py:475`. This is injected into every initial message. May contain test-related instructions.
   - Look at: `prompt_template.txt` in the repo root

4. **`claude_code.py` initial message (lines 1232-1327)** — The test format examples we changed show `shared`, `infrastructure`, `backend`, `functional`, `style` categories. But check if the agent is actually seeing these examples or if something else overwrites them. The `create_thyme_style_message()` call at line 1192 builds the base message from BUILD_PLAN, then lines 1232+ append the test instructions. Verify the final assembled message includes the backend test examples.

5. **`prompts/canopy/BUILD_PLAN.md`** — The build plan itself may describe features in a way that only suggests frontend tests. If the test section of the build plan lists UI acceptance criteria but no backend test criteria, the agent will write UI tests.
   - Look at: `prompts/canopy/BUILD_PLAN.md` — search for any test-related sections

6. **The `playwright-test.cjs` helper** — The test verification workflow is entirely screenshot-based (take screenshot, view it, mark passing). This workflow only makes sense for frontend tests. There's no equivalent "run this backend test and verify" workflow described. The agent may be writing only tests it knows how to verify.
   - Consider: Should we add a backend test verification workflow? e.g., `cd backend && npm test` or `cd shared && npx tsc --noEmit` as verifiable test steps?

**Other remaining issues:**
- All 220 tests are marked `"passes": false` in the final tests.json despite the progress file claiming they all pass — the batch-marking commit may have had issues
- A separate QA agent is planned to handle thorough end-to-end testing

**Files changed in the last session (commit `8dfdae7` on `kb/improved-harness`):**
- `prompts/system_prompt.txt` — Testing/Quality, Test Verification, Signaling Completion sections
- `prompts/canopy/system_prompt.txt` — Same + added Phased Execution section + @canopy/shared imports
- `claude_code.py` — Initial message (lines ~1232-1327) and continuation message (line ~1416)

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
