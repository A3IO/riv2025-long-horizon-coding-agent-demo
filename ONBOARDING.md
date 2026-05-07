# Developer Onboarding Guide

Welcome to the team. This doc is meant to be your guided tour — the kind of thing a senior engineer would walk you through on your first week. It references specific files and functions rather than abstractions, because that's what actually helps.

---

## 1. Project Overview

**What does this project do?**

This is an autonomous coding agent that builds full-stack web applications from GitHub issues. You create a GitHub issue, a team admin approves it with a rocket (🚀) reaction, and the agent — running in an AWS container — clones the repo, writes all the code, commits it, and deploys a live preview. No human writes a single line of the generated app.

**What problem does it solve?**

It demonstrates long-horizon agentic software engineering: an AI agent that can independently navigate a multi-hour, multi-phase development workflow (infrastructure → backend → frontend → tests → deployment) without human intervention, using real dev tools (Git, CDK, Playwright, DynamoDB, etc.).

**Who are the end users?**

Two audiences: (1) the **operator** — the person who configures `PROJECT_NAME`, pushes the Docker image, and creates GitHub issues to trigger builds; and (2) attendees of demos (this was shown at re:Invent 2025) who watch the agent build a real app live.

---

## 2. Tech Stack

### Harness (the system that runs the agent)

| Layer | Tech |
|---|---|
| Container runtime | AWS Bedrock AgentCore (Graviton, ARM64) |
| Agent orchestration | Python 3.12, `claude-agent-sdk`, `bedrock-agentcore` |
| AI model | `us.anthropic.claude-opus-4-6-v1` via Amazon Bedrock |
| Observability | AWS Distro for OpenTelemetry (ADOT), CloudWatch Logs |
| Infrastructure | AWS CDK (TypeScript), CloudFormation |
| Container registry | Amazon ECR |
| Secrets | AWS Secrets Manager |
| State coordination | AWS SSM Parameter Store |
| CI/CD | GitHub Actions (5 workflows) |
| Frontend hosting | S3 + CloudFront |
| Testing (harness E2E) | Playwright |

### Generated App (the Canopy project the agent builds)

| Layer | Tech |
|---|---|
| Frontend | React 18/19, Vite 6/7, Tailwind CSS v4, React Router v7 |
| Backend | AWS Lambda (Node.js 20), API Gateway HTTP API |
| Database | Amazon DynamoDB (single-table design) |
| Shared types | Zod schemas (`shared/` package — source of truth) |
| Build bundler | esbuild (via CDK `NodejsFunction`) |
| UI components | shadcn/ui, Radix UI, lucide-react |
| Drag and drop | @dnd-kit |
| Charts | Recharts |

### Python Dependencies (`requirements.txt`)

```
claude-agent-sdk>=0.1.6      # Core: runs Claude agent sessions
boto3>=1.28.0                # AWS SDK (CloudWatch, SSM, Secrets Manager)
PyGithub>=2.8.1              # GitHub issue/PR management
opentelemetry-api>=1.20.0    # OTEL trace propagation
aws-opentelemetry-distro     # AWS ADOT for CloudWatch delivery
python-dotenv>=1.0.0
diagrams>=0.23.0             # Architecture diagrams
```

---

## 3. Architecture & Structure

### High-Level Pattern

This is a **serverless event-driven pipeline** at the harness level, and produces a **serverless monorepo app** as output.

```
GitHub Issue (🚀 approved)
        │
        ▼
GitHub Actions (issue-poller.yml → agent-builder.yml)
        │
        ▼
AWS Bedrock AgentCore (Docker container, ARM64 Graviton)
        │
        ▼
bedrock_entrypoint.py  ──────►  claude_code.py
   (orchestrator)               (agent loop)
        │                            │
        │                     Claude SDK ↔ Bedrock
        │                            │
        │                     Writes code to disk
        │                            │
        ▼                            ▼
Git post-commit hook         Security validation hooks
   (auto-push)               (bash_security_hook,
        │                     universal_path_security_hook)
        ▼
agent-runtime branch (GitHub)
        │
        ▼
deploy-preview.yml (GitHub Actions)
        │
        ▼
S3 + CloudFront (live preview URL posted to issue)
```

### Directory Structure

```
.
├── bedrock_entrypoint.py          # Entry point: orchestrates everything
├── claude_code.py                 # 2000-line agent session manager
├── Dockerfile                     # ARM64 container definition
├── Makefile                       # All management commands
├── requirements.txt               # Python deps
├── prompt_template.txt            # Template variable substitution
├── state_management.txt           # Reference doc for agent state machine
│
├── src/                           # Core harness modules
│   ├── config.py                  # Constants: model, ports, security allowlists
│   ├── security.py                # Pre/post tool hooks — the enforcement layer
│   ├── git_manager.py             # Git: clone, branch, hooks, push
│   ├── github_integration.py      # GitHub API: issues, labels, comments
│   ├── session_manager.py         # Session setup, prompt file management
│   ├── token_tracker.py           # Token usage tracking and cost limits
│   ├── logging_utils.py           # Structured logging with timestamps
│   └── cloudwatch_metrics.py      # Heartbeat metrics publishing
│
├── prompts/                       # Baked into Docker image at build time
│   ├── system_prompt.txt          # Base system prompt (all projects)
│   ├── DEBUGGING_GUIDE.md         # General debugging guidance
│   ├── FRONTEND_AESTHETICS_GUIDE.md  # UI design anti-patterns
│   └── canopy/                    # Project-specific prompt files
│       ├── BUILD_PLAN.md          # Full app specification (the "brief")
│       ├── EXAMPLE_TEST.txt       # Test patterns to follow
│       ├── DEBUGGING_GUIDE.md     # Canopy-specific debug tips
│       └── system_prompt.txt      # Canopy-specific prompt additions
│
├── frontend-scaffold-template/    # Pre-configured React+Vite starter
│   ├── package.json               # 89 dependencies pre-listed
│   ├── vite.config.ts             # HMR configured for HTTPS behind ALB
│   ├── src/api/client.ts          # Fetch-based API client
│   └── src/hooks/use-mobile.ts    # Mobile breakpoint hook
│
├── infrastructure/                # CDK stack for the harness infrastructure
│   ├── bin/claude-code-infrastructure.ts  # CDK app entry point
│   ├── lib/claude-code-stack.ts   # ECR, EFS, CloudFront, IAM, S3
│   └── cdk.json                   # CDK toolkit config
│
├── .github/
│   ├── workflows/
│   │   ├── issue-poller.yml       # Cron: find approved issues every 5 min
│   │   ├── agent-builder.yml      # Dispatch: invoke AgentCore session
│   │   ├── deploy-preview.yml     # Push hook: build + deploy to CloudFront
│   │   ├── deploy-infrastructure.yml  # Deploy CDK stack changes
│   │   └── stop-agent-on-close.yml    # Kill session when issue is closed
│   └── scripts/invoke_agent.py    # Boto3 script to start AgentCore session
│
└── e2e-tests/
    └── smoke.spec.ts              # Playwright smoke tests (harness-level)
```

### How Data Flows (Request Lifecycle)

1. **Trigger**: GitHub issue + 🚀 reaction detected by `issue-poller.yml` cron
2. **Dispatch**: `agent-builder.yml` invokes AgentCore via `invoke_agent.py`, passing JSON payload with issue number, title, body, and session ID
3. **Container init**: `bedrock_entrypoint.py` starts, fetches GitHub token from Secrets Manager, clones the harness repo, sets up SSH, installs post-commit git hook
4. **Agent loop**: `claude_code.py` loads `BUILD_PLAN.md` + system prompts, sends initial message to Claude via Agent SDK, then runs an async loop processing tool calls
5. **Security enforcement**: Every bash command passes through `bash_security_hook` in `src/security.py`; every file operation passes through `universal_path_security_hook`
6. **Commits**: Agent makes commits → post-commit hook auto-pushes to `agent-runtime` branch → `deploy-preview.yml` triggers → CloudFront updates
7. **Test verification**: Agent must Read a screenshot file (or `-result.txt` for backend tests) before marking any test as passing — enforced in `src/security.py:_validate_test_result_modification()`
8. **Completion**: Agent outputs "🎉 implementation complete" → `claude_code.py` detects signal → `bedrock_entrypoint.py` posts to GitHub issue, marks complete, closes issue

### Key Design Patterns

- **Hook-based security**: Every Claude SDK tool call goes through pre/post hooks (`PreToolUse`, `PostToolUse`) defined in `src/security.py`. This is the single enforcement layer — no other code path can bypass it.
- **State machine**: Agent has 5 states (`continuous`, `run_once`, `run_cleanup`, `pause`, `terminated`) managed via `agent_state.json`. `claude_code.py:read_agent_state()` / `write_agent_state()` manage transitions.
- **Phase-driven prompts**: The system prompt structures work into phases (Phase 2a CDK + stubs → wait for CI/CD → Phase 2b full handlers → Phase 3 frontend). The agent is incentivized by phase completion, not by instructions alone.
- **Incentives over instructions**: The most important design lesson in this codebase. See Section 10.

---

## 4. Key Files & Entry Points

### Where Does It Start?

**Container startup** → `Dockerfile` CMD → `bedrock_entrypoint.py`

The CMD in the Dockerfile copies prompts and scaffold template to the workspace, then runs:
```
opentelemetry-instrument python bedrock_entrypoint.py
```

The `opentelemetry-instrument` wrapper is critical — without it, ADOT doesn't capture Python `logging` output, and you get no CloudWatch logs.

### The 10 Files to Read First

| Priority | File | Why |
|---|---|---|
| 1 | `bedrock_entrypoint.py` | The orchestrator — understand the container startup and GitHub mode setup |
| 2 | `claude_code.py` | The 2000-line brain — agent session loop, state machine, message construction |
| 3 | `src/security.py` | The enforcement layer — understand this before touching anything else |
| 4 | `src/config.py` | All constants: model, ports, command allowlists, blocked patterns |
| 5 | `prompts/canopy/BUILD_PLAN.md` | What the agent is actually being asked to build |
| 6 | `prompts/system_prompt.txt` | Agent instructions: phase ordering, CDK-first flow, naming conventions |
| 7 | `src/git_manager.py` | Git operations: post-commit hook, push, branch setup |
| 8 | `.github/workflows/agent-builder.yml` | How GitHub Actions invokes AgentCore |
| 9 | `infrastructure/lib/claude-code-stack.ts` | CDK stack: ECR, EFS, CloudFront, IAM |
| 10 | `Makefile` | All management commands and their environment variable defaults |

### Main Configuration Files

| File | Controls |
|---|---|
| `Makefile` | Runtime ID, execution role ARN, VPC ID, model, project name, push interval, ECR URI |
| `src/config.py` | Model default, port defaults, bash command allowlists, blocked patterns |
| `infrastructure/lib/claude-code-stack.ts` | All AWS resources (CDK context variables) |
| `.github/workflows/agent-builder.yml` | AgentCore runtime ID, AWS region, concurrency rules |
| `Dockerfile` | Container image definition, non-root user setup, tool installations |

---

## 5. Core Domain Concepts

### Main Entities

**AgentCore Runtime** — The AWS-managed container environment. Identified by its runtime ID (set as `AGENT_RUNTIME_ID` in `Makefile.local`). Each session is a separate invocation. The runtime holds the Docker image and environment variables. Think of it as a managed EC2 that AWS provisions per-request.

**Session** — A single agent run, identified by a 33-character session ID. Lives for up to 8 hours. One session builds one issue. Tracked in SSM: `/claude-code/session-id`.

**Issue** — A GitHub issue represents a build request. Must have a 🚀 reaction from an authorized approver to be eligible. Tracked through states: open → `agent-building` label → `agent-complete` label → closed.

**Project** — A named set of prompts under `prompts/{project-name}/`. The only required file is `BUILD_PLAN.md`. Currently only `canopy` exists. Set via `PROJECT_NAME` env var.

**agent-runtime branch** — The Git branch where generated code lives. The agent clones the base branch (`main`), checks out `agent-runtime`, and commits its work there. This branch is wiped on `make reset`.

**deploy-state** — An SSM parameter at `/claude-code/infra/deploy-state`. JSON blob written by the `deploy-infrastructure.yml` workflow after CDK deploys succeed. Contains `ApiUrl`, `FrontendBucketName`, `DistributionId`, `status`. The agent polls this to know when infrastructure is ready.

**tests.json** — A file the agent maintains in the generated app's root. Tracks which tests pass/fail. The security layer aggressively protects this file — the agent cannot modify it via bash commands, only via the Edit tool, and only after reading screenshot evidence.

### Critical Flows

**Phase flow (CDK-first)**:
1. Phase 1: Set up monorepo, shared Zod schemas, `tests.json`
2. Phase 2a: Write CDK stack + stub Lambda handlers, commit + push (triggers `deploy-infrastructure.yml`)
3. **Wait**: Poll SSM `deploy-state` until `status=succeeded`
4. Phase 2b: Read `ApiUrl` from deploy-state, implement full Lambda handlers
5. Phase 3: Create `frontend/.env` with `VITE_API_URL`, build full frontend
6. Phase 4: Write and verify tests (Playwright for frontend, `backend-verify.cjs` for backend)

**Test verification flow** (enforced by `src/security.py:_validate_test_result_modification()`):
- Frontend test: Agent must run Playwright, which saves `screenshots/test-id.png` + `screenshots/test-id-console.txt` → agent must `Read` both files → then can mark test passing in `tests.json`
- Backend test: Agent runs `backend-verify.cjs` which writes `screenshots/test-id-result.txt` (must contain `VERIFIED_BY: backend-verify.cjs` + `RESULT: PASS`) + `screenshots/test-id-console.txt` → agent must `Read` both → then can mark passing

### Terminology

- **harness** — This codebase. The system that runs the agent. Not the app being built.
- **generated app** — The code the agent writes (Canopy). Lives on `agent-runtime` branch.
- **scaffold template** — `frontend-scaffold-template/`. Pre-configured React+Vite project the agent copies as a starting point.
- **build plan** — `prompts/canopy/BUILD_PLAN.md`. The full specification the agent reads and implements.
- **phase completion** — How the agent is incentivized. Each phase has a clear deliverable that unlocks the next phase. See `prompts/system_prompt.txt`.
- **deploy-state** — The SSM parameter that bridges CDK deployment (CI/CD) with the agent's awareness of what's deployed.

---

## 6. API / Interface Surface

### This Is Not a Traditional API Project

The harness itself doesn't expose HTTP endpoints. Its "interface" is:

1. **AWS Bedrock AgentCore invocation** — `invoke_agent.py` calls `bedrock_agentcore.runtime.invoke_agent()` with a JSON payload
2. **GitHub webhooks/polling** — `issue-poller.yml` polls GitHub API for approved issues every 5 minutes
3. **SSM parameters** — Agent reads/writes state via SSM:
   - `/claude-code/current-issue` — current issue number being built
   - `/claude-code/session-id` — active session ID
   - `/claude-code/infra/deploy-state` — deployment status JSON from CDK workflow
4. **Git** — The agent's "output interface" is commits to the `agent-runtime` branch

### The Generated App's API (Canopy)

The app the agent builds exposes a REST API via API Gateway + Lambda. From `BUILD_PLAN.md`, expected endpoints include:

- `POST /projects`, `GET /projects`, `GET /projects/{id}`, `PATCH /projects/{id}`
- `POST /projects/{id}/issues`, `GET /projects/{id}/issues`, `PATCH /issues/{id}`
- `POST /projects/{id}/sprints`, `PATCH /sprints/{id}`
- `POST /issues/{id}/comments`

All request/response types are defined as Zod schemas in the generated `shared/` package.

### Authentication

The harness uses **AWS IAM** throughout:
- AgentCore container uses `claude-code-agentcore-role` IAM role
- GitHub Actions uses OIDC with a dedicated IAM role (configured in CDK stack)
- The generated app uses API Gateway without auth (it's a demo app)

---

## 7. Development Workflow

### Prerequisites

- AWS CLI configured with `default` profile, access to your AWS account
- Docker with `--platform linux/arm64` build support (or Rosetta on Apple Silicon)
- Node.js + npm (for CDK and infrastructure)
- `gh` CLI authenticated to GitHub
- Python 3.12+

### Local Dev (Running Agent Locally)

```bash
# Install Python dependencies
pip install -r requirements.txt

# Run agent locally against a project
make launch-local PROJECT_NAME=canopy

# Or run directly
python claude_code.py --project canopy
```

Note: Local mode doesn't have GitHub integration or AgentCore. It runs the Claude SDK directly.

### Full Cloud Deployment

```bash
# 1. Deploy CDK infrastructure (only needed if infra changed)
make deploy-infra

# 2. Build and push Docker image (required whenever code/prompts change)
make show-config    # Get ECR_URI
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <ECR_URI>
docker build --platform linux/arm64 -t <ECR_URI>:latest .
docker push <ECR_URI>:latest

# 3. Update runtime environment variables
make update-runtime-env

# 4. Reset state for a fresh run
make reset

# 5. Create an issue and trigger the agent
gh issue create --repo YOUR_ORG/YOUR_REPO \
  --title "[MVP] Canopy Build" \
  --body "Build the Canopy app as specified in BUILD_PLAN.md."
gh api repos/YOUR_ORG/YOUR_REPO/issues/ISSUE_NUM/reactions \
  -f content=rocket
make trigger ISSUE_NUM=N  # or use gh workflow run
```

### Monitoring a Live Run

```bash
# Watch CloudWatch logs (last 5 minutes)
aws logs filter-log-events \
  --log-group-name "/aws/bedrock-agentcore/runtimes/<YOUR_RUNTIME_ID>-DEFAULT" \
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

# Check agent-runtime commits
gh api "repos/YOUR_ORG/YOUR_REPO/commits?sha=agent-runtime&per_page=5" \
  --jq '.[] | .sha[:8] + " " + .commit.author.date + " " + (.commit.message | split("\n")[0])'

# Stop a stuck session
make stop-session SESSION_ID=<id-from-issue-comment>
```

### Environment Variables

| Variable | Required | Secret? | Description |
|---|---|---|---|
| `PROJECT_NAME` | Yes | No | Which project to build (`canopy`) |
| `BASE_BRANCH` | Yes | No | Harness code branch (`main`) |
| `CLAUDE_CODE_USE_BEDROCK` | Yes | No | Set to `1` to use Bedrock endpoint |
| `AWS_REGION` | Yes | No | `us-east-1` |
| `DEFAULT_MODEL` | No | No | Model ID override |
| `PUSH_INTERVAL_SECONDS` | No | No | How often to push commits (default: 300) |
| `SCREENSHOT_INTERVAL_SECONDS` | No | No | Screenshot capture interval |
| `SESSION_DURATION_HOURS` | No | No | Max session length |
| `AGENT_RUNTIME_ID` | Yes | No | AgentCore runtime ID |
| `EXECUTION_ROLE_ARN` | Yes | No | IAM role for the container |
| `GITHUB_TOKEN` | Yes | **Secret** | Fetched from Secrets Manager at runtime |
| `ANTHROPIC_API_KEY` | Conditional | **Secret** | Only needed for local runs (not Bedrock) |
| `CLAUDE_CODE_ENABLE_TELEMETRY` | No | No | Set to `1` for OTEL telemetry |
| `AGENT_OBSERVABILITY_ENABLED` | No | No | Set to `true` for AgentCore observability |

**Warning**: `make update-runtime-env` **replaces all environment variables** on the AgentCore runtime. If you add a new variable anywhere but forget to add it to the `update-runtime-env` Makefile target, it gets silently wiped. This caused a cascade of failures (issues #5-#13). Always add new vars to both `launch` and `update-runtime-env`.

---

## 8. Testing

### Testing Strategy

The testing is multi-layered and somewhat unusual:

1. **Harness-level E2E** (`e2e-tests/smoke.spec.ts`): Playwright tests that verify the deployed preview URL is reachable. Run via `npm test:e2e`.

2. **Generated app tests** (written by the agent): The agent writes its own tests as part of building the app. There are two verification mechanisms:
   - **Playwright frontend tests**: Run via `playwright-test.cjs`, capture screenshots + console logs to `screenshots/`
   - **Backend tests**: Run via `backend-verify.cjs`, which shells out to `curl`/`aws` CLI commands and writes `-result.txt` + `-console.txt` to `screenshots/`

3. **CDK infrastructure tests** (generated by agent): Jest tests in `infrastructure/test/` that run `cdk synth` and check CloudFormation output.

### Where Tests Live

- `e2e-tests/` — Harness smoke tests
- `playwright.config.staging.ts` — Playwright config pointing at staging CloudFront URL
- The agent writes tests into the generated app on `agent-runtime` branch (not committed to this repo)

### Test Verification Enforcement (`src/security.py`)

This is the most unusual testing aspect: the agent cannot self-report passing tests. Every test result is enforced by `_validate_test_result_modification()`:

- The agent must run the test runner first (Playwright or `backend-verify.cjs`)
- These runners write artifacts to `screenshots/`
- The agent must `Read` those artifact files (tracked by `track_screenshot_read()`)
- Only then can it use the Edit tool to mark the test as passing in `tests.json`

If the agent tries to mark a test passing without reading the screenshot, the hook returns a deny response with an explanation.

### Current Coverage Observations

From the most recent successful run (issue #22): **54/54 tests passing** — 3 shared, 7 infrastructure, 6 backend, 38 frontend. Frontend is the most thoroughly tested area; shared/infrastructure are under-tested. The agent tends to write more frontend tests when given freedom.

---

## 9. Patterns & Conventions

### Naming Conventions

- **All generated AWS resources must use `canopy-` prefix** (`canopy-projects-table`, `canopy-create-project`, etc.) — IAM policies are scoped to `canopy-*` so anything without the prefix will get permission errors at runtime
- **CDK stack name**: `canopy-app-stack`
- **Python**: snake_case for functions and variables, PascalCase for classes
- **TypeScript/React**: PascalCase for components, camelCase for functions
- **Git branches**: `agent-runtime` (generated app), `issue-N` (alternative per-issue branches), `kb/improved-harness` (active harness branch)

### Logging Strategy

**Critical**: `print()` is NOT captured by CloudWatch when running in AgentCore. Only Python `logging` module output reaches CloudWatch via OTEL auto-instrumentation.

In `bedrock_entrypoint.py`:
```python
logging.basicConfig(level=logging.INFO, format='%(asctime)s [%(name)s] %(levelname)s %(message)s')
logger = logging.getLogger("agent.entrypoint")
# Use: logger.info("...") NOT print("...")
```

The logger name `agent.entrypoint` appears in CloudWatch events and is used in the monitoring commands above to filter relevant log entries.

`src/logging_utils.py:LoggingManager` wraps `print()` in `claude_code.py` to add timestamps and write to a log file, but this only applies within the agent session loop, not in `bedrock_entrypoint.py`.

### Error Handling

- Most AWS/GitHub operations use try/except with fallback behavior. If CloudWatch metrics fail, the container continues. If the GitHub token fetch fails, the container exits with an error.
- The agent detects error conditions by string matching in tool output: "prompt is too long", "JSON buffer size", "image size error" are treated as terminal conditions in `claude_code.py:_detect_completion_signal()`.
- Security hook denials use `_deny_response()` in `src/security.py`, which returns a structured dict that the Claude SDK presents back to the agent as an error message, allowing the agent to try a different approach.

### Code Style / Linting

No linting config found in this repo. The Python code uses type hints inconsistently (present in some files, absent in others). The `src/` modules are generally more type-annotated than the root-level scripts.

### Shared Utilities

- `src/config.py:ALLOWED_BASH_COMMANDS` — the canonical allowlist; add new commands here
- `src/security.py:_deny_response()` — standardized hook denial format
- `src/git_manager.py:GitManager` — always use this for git operations, not raw subprocess calls
- `src/github_integration.py:GitHubIssueManager` — all GitHub API calls go through this
- `bedrock_entrypoint.py:get_secret()` — use for all Secrets Manager fetches

---

## 10. Gotchas & Tribal Knowledge

These are the real things that will trip you up. Read this section carefully.

### 1. `update-runtime-env` nukes all env vars

The Makefile `update-runtime-env` target calls `update-agent-runtime` with the full env var list. If you add a new env var to `launch` but forget to add it to `update-runtime-env`, it silently disappears on the next update. This was the root cause of issues #5 through #13, where `CLAUDE_CODE_USE_BEDROCK=1` was being wiped repeatedly. **Always keep both targets in sync.**

### 2. `print()` is invisible in CloudWatch

Use `logger.info()` in `bedrock_entrypoint.py`. Only the Python `logging` module is hooked by OTEL auto-instrumentation. `print()` statements in `bedrock_entrypoint.py` write to stdout but don't appear in CloudWatch. This makes debugging production issues very hard if someone adds `print()` calls.

### 3. Docker images must be ARM64

AgentCore runs on AWS Graviton. Always build with:
```bash
docker build --platform linux/arm64 -t <ECR_URI>:latest .
```
An x86_64 image will appear to push fine, then silently fail to start in production.

### 4. `bypassPermissions` requires non-root user

The Claude Code CLI refuses `--dangerously-skip-permissions` when running as root. The Dockerfile creates a non-root `agent` user (UID 1000) specifically for this. Don't change the `USER agent` line in the Dockerfile.

### 5. Prompts are baked into the Docker image

The `Dockerfile` has `COPY prompts/ ./prompts/`. This means **any prompt change requires a Docker image rebuild and push**. Just merging to `main` does nothing. See the Deploying Changes table in `CLAUDE.md` for what requires a rebuild vs. not.

### 6. CDK `agentCoreRoleName` must match exactly

The container execution role is `claude-code-agentcore-role`. If you deploy CDK with the wrong `agentCoreRoleName`, IAM policies attach to the wrong role (the auto-created `AmazonBedrockAgentCoreSDKRuntime-*` role). The container will start but **crash silently** because it can't access Secrets Manager, SSM, or CloudWatch. You'll see zero log entries after the initial handler init. Always use `make deploy-infra` which passes the correct defaults.

### 7. The agent takes 20-30 minutes before its first commit

Don't panic. The agent builds up a large batch of files before its first commit. The post-commit hook and background push loop handle pushing once commits exist. If you see no commits after 30+ minutes, check CloudWatch logs for errors.

### 8. Incentives shape behavior more than instructions

This is the biggest architectural lesson. When the prompt said "graded on UI quality" and "200 tests", the agent built a frontend-only app with 200+ frontend tests and skipped infrastructure entirely. Replacing grading language with phase-completion scoring fixed the build order immediately. If the agent is doing the wrong thing, **change the incentive structure**, not just the instructions.

### 9. CDK deploy without `vpcId` creates a new VPC

The AWS account has a VPC limit. Always pass `-c vpcId=<YOUR_VPC_ID>` (or set `VPC_ID` in `Makefile.local` and use `make deploy-infra` which passes it automatically). Creating an extra VPC by accident is annoying to clean up.

### 10. f-string escaping in `bedrock_entrypoint.py`

When `bedrock_entrypoint.py` constructs JSON strings inside Python f-strings, curly braces must be double-escaped (`{{` and `}}`). This has caused bugs. Be careful when modifying that file.

### 11. `sed`/`awk`/`jq` on `tests.json` are blocked

The security layer blocks any bash command that touches `tests.json` via `sed`, `awk`, `jq`, `python3`, redirection, etc. This is intentional — the agent must use the Edit tool with screenshot verification. If you're debugging test failures and wondering why the agent can't "just fix" the JSON, that's why.

### 12. The CDK-first deployment flow is NOT YET VALIDATED (as of last CLAUDE.md update)

The prompt changes that enforce "wait for CDK deploy before writing handlers" were committed in `3744400` but hadn't been tested in a live run as of `2026-02-20`. The prompt changes look right, but there may be edge cases. Monitor the next live run closely for SSM polling activity in CloudWatch.

### 13. `agent-runtime` branch is ephemeral

`make reset` deletes `agent-runtime` both locally and remotely. Don't put anything you care about on that branch. It exists purely as the output target for the current build.

---

## 11. Dependency Map

### Internal Module Dependencies

```
bedrock_entrypoint.py
  ├── src/git_manager.py (GitManager, GitHubConfig)
  ├── src/github_integration.py (GitHubIssueManager)
  └── src/cloudwatch_metrics.py (MetricsPublisher)

claude_code.py
  ├── src/security.py (bash_security_hook, universal_path_security_hook, track_read_hook)
  ├── src/config.py (ALL constants)
  ├── src/token_tracker.py (TokenTracker)
  ├── src/logging_utils.py (LoggingManager)
  └── src/session_manager.py (get_project_prompts_dir)

src/security.py
  └── src/config.py (ALLOWED_BASH_COMMANDS, BLOCKED_*_PATTERNS, etc.)

src/git_manager.py
  └── (no internal deps — only stdlib + boto3)

src/github_integration.py
  └── (no internal deps — only PyGithub + stdlib)
```

### Critical Third-Party Dependencies

| Package | Version | Purpose | Notes |
|---|---|---|---|
| `claude-agent-sdk` | `>=0.1.6` | Core agent loop, tool invocation | The SDK that runs Claude; hooks are registered here |
| `bedrock-agentcore` | latest | AWS Bedrock AgentCore runtime | `BedrockAgentCoreApp`, `PingStatus` |
| `boto3` | `>=1.28.0` | All AWS API calls | CloudWatch, SSM, Secrets Manager, CloudFormation |
| `PyGithub` | `>=2.8.1` | GitHub issue management | Issues, labels, comments, reactions |
| `aws-opentelemetry-distro` | `>=0.13.0` | ADOT for CloudWatch log delivery | Required for `logger.info()` to appear in CloudWatch |
| `opentelemetry-api` | `>=1.20.0` | Session ID trace propagation | |
| `@anthropic-ai/claude-code` | (npm, global) | Claude Code CLI | Required by claude-agent-sdk; installed in Dockerfile |
| `aws-cdk` | (npm, global) | CDK CLI for `cdk synth/test` | Installed in Dockerfile; agent uses for infra tests |

### Pinned/Compatibility Notes

- **Python 3.12** is pinned in the Dockerfile base image. The `match` statement in `src/security.py` requires Python 3.10+.
- **ARM64** is required for AgentCore (Graviton). The Dockerfile detects architecture for the AWS CLI v2 download.
- **Node.js** is installed via apt in the Dockerfile (old version from Debian repos). The `frontend-scaffold-template` uses features compatible with Node 18+.
- `claude-agent-sdk` version is the most likely to cause breaking changes — it's still early-stage software. Check for breaking API changes before upgrading.

---

## 12. Deployment & Infrastructure

### CI/CD Pipeline

There are 5 GitHub Actions workflows:

| Workflow | Trigger | What It Does |
|---|---|---|
| `issue-poller.yml` | Cron every 5 min | Polls GitHub for approved issues; detects stale sessions; triggers `agent-builder.yml` |
| `agent-builder.yml` | `workflow_dispatch` | Checks approval, acquires lock, invokes AgentCore session via `invoke_agent.py` |
| `deploy-preview.yml` | Push to `agent-runtime` | Builds the generated app, deploys to S3/CloudFront, posts URL to issue |
| `deploy-infrastructure.yml` | Push to `agent-runtime` | Runs CDK synth/test/deploy; writes `deploy-state` to SSM; triggers frontend build |
| `stop-agent-on-close.yml` | Issue closed | Stops running AgentCore session |

### Environment Tiers

There is effectively **one environment**: `reinvent` (production). There's no staging or dev tier. Local runs use a different code path (`make launch-local`) that doesn't use AgentCore.

The `ENVIRONMENT` variable in the Makefile sets the CloudFormation stack name suffix and affects secret paths in Secrets Manager.

### Infrastructure Resources (CDK Stack `claude-code-reinvent`)

Defined in `infrastructure/lib/claude-code-stack.ts`:

- **ECR Repository**: `<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/<projectName>-<environment>` — Docker images for the agent
- **EFS File System**: Persistent storage for agent workspace (survives container restarts)
- **Secrets Manager**: GitHub token and other secrets
- **CloudWatch Log Group**: `/aws/bedrock-agentcore/runtimes/<YOUR_RUNTIME_ID>-DEFAULT`
- **S3 Buckets**: Screenshots/CDN bucket + deploy-preview bucket
- **CloudFront**: Two distributions — one for screenshots CDN, one for the generated app preview
- **IAM Roles**: `claude-code-agentcore-role` (container) + GitHub Actions OIDC role

### CDK Context Variables

Override when deploying with `cdk deploy -c key=value`:

| Variable | Default | Description |
|---|---|---|
| `projectName` | `claude-code` | Prefix for AWS resource names |
| `environment` | `reinvent` | Stack name suffix + secret path suffix |
| `vpcId` | (creates new — **don't do this**) | Use existing VPC |
| `agentCoreRoleName` | (required) | Must be `claude-code-agentcore-role` |
| `agentRuntimeId` | `YOUR_AGENT_RUNTIME_ID` | For log group naming |
| `logRetentionDays` | `7` | CloudWatch retention |

**Always use `make deploy-infra`** which passes the correct `vpcId` and `agentCoreRoleName` automatically. Running `cdk deploy` directly without context variables will create a new VPC and attach policies to the wrong role.

### Resetting Everything

```bash
make reset
```

This deletes: `agent-runtime` branch (local + remote), all open `agent-building` issues, SSM parameters (`/claude-code/current-issue`, `/claude-code/session-id`, `/claude-code/infra/deploy-state`), empties S3 buckets, invalidates CloudFront caches.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         GitHub                                       │
│  Issue #N ──[🚀 reaction]──► issue-poller.yml (cron 5min)           │
│                                      │                               │
│                              agent-builder.yml                       │
│                                      │                               │
└──────────────────────────────────────┼──────────────────────────────┘
                                       │ workflow_dispatch
                                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    AWS (us-east-1)                                    │
│                                                                      │
│  invoke_agent.py ──► Bedrock AgentCore Runtime                       │
│                           │                                          │
│                    ┌──────▼──────────────────┐                       │
│                    │  Docker Container (ARM64) │                      │
│                    │  bedrock_entrypoint.py   │                       │
│                    │         │                │                       │
│                    │  claude_code.py          │                       │
│                    │    ├─ security.py        │                       │
│                    │    ├─ git_manager.py     │                       │
│                    │    └─ github_integration │                       │
│                    └──────────────────────────┘                      │
│                           │                    │                     │
│                    Secrets Manager         CloudWatch Logs            │
│                    SSM Parameters          (OTEL/ADOT)                │
│                    ECR (image)                                        │
│                                                                      │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ git push to agent-runtime
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  GitHub: agent-runtime branch                                        │
│    ├─ deploy-preview.yml ──► S3 + CloudFront (preview URL)           │
│    └─ deploy-infrastructure.yml ──► CDK deploy ──► SSM deploy-state  │
└─────────────────────────────────────────────────────────────────────┘
```

---

*This document was generated from the codebase as of `2026-02-25` on branch `main`. If you find something that doesn't match reality, the code is ground truth — update this doc.*
