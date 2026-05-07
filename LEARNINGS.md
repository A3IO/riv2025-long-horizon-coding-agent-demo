# Deployment Learnings: Adapting the Long-Horizon Coding Agent to a New AWS Account

This document captures every issue encountered and lesson learned when deploying this
project from scratch to a new AWS account and GitHub repository. It is written to be
fully generalizable — all account numbers, resource IDs, and repo names are replaced
with placeholders. Use it as a checklist when forking for a new deployment.

**Placeholder convention used throughout:**
| Placeholder | Meaning |
|-------------|---------|
| `<ACCOUNT_ID>` | Your AWS account number (12 digits) |
| `<REGION>` | Your target AWS region (e.g. `us-east-1`) |
| `<RUNTIME_ID>` | Your AgentCore runtime ID (e.g. `claude_code_reinvent-XXXXXXXXXX`) |
| `<ROLE_NAME>` | Your AgentCore execution role name (e.g. `claude-code-agentcore-role`) |
| `<VPC_ID>` | Your VPC ID (e.g. `vpc-XXXXXXXXXXXXXXXXX`) |
| `<GH_OWNER>/<GH_REPO>` | Your target GitHub repo (e.g. `myorg/myapp`) |
| `<ECR_URI>` | Your ECR repository URI (from `make show-config`) |

---

## Part 1: Setup Friction Points

Issues encountered during initial environment setup, CDK deployment, and infrastructure
configuration.

---

### 1. `agentcore` CLI is broken (bedrock-agentcore PyPI package)

**What happened:** `pip install bedrock-agentcore` installs the SDK package, which
registers an `agentcore` CLI entrypoint pointing at `bedrock_agentcore.cli` — a module
that does not exist.

```
ModuleNotFoundError: No module named 'bedrock_agentcore.cli'
```

**Workaround:** Use the AWS CLI directly for all runtime management:
```bash
aws bedrock-agentcore-control create-agent-runtime ...
aws bedrock-agentcore-control update-agent-runtime ...
aws bedrock-agentcore-control get-agent-runtime ...
```

The Makefile targets (`make deploy-infra`, `make update-runtime-env`, etc.) already use
the AWS CLI, so this only affects ad-hoc commands.

---

### 2. `infrastructure/package-lock.json` references an internal registry

**What happened:** The checked-in `package-lock.json` had `resolved` URLs pointing at an
internal npm registry. Running `npm install` (even with `--registry https://registry.npmjs.org`)
fails with HTTP 401 because the lock file's `resolved` fields take precedence.

**Fix:** Delete `package-lock.json` and regenerate it against the public registry:
```bash
rm infrastructure/package-lock.json
cd infrastructure && npm install
```

---

### 3. Shell `AWS_REGION` overrides CDK region

**What happened:** A shell session had `AWS_REGION` exported to a non-target region.
CDK picked it up and tried to deploy there, failing with VPC-not-found errors.

**Fix:** Always override explicitly when running CDK commands:
```bash
make deploy-infra AWS_REGION=<REGION>
```
Or unset the shell variable first: `unset AWS_REGION`.

See also item **L** (Makefile `?=` caveat) and item **M** (global export).

---

### 4. CDK bootstrap required on first deploy

**What happened:** First CDK deploy failed with:
```
SSM parameter /cdk-bootstrap/hnb659fds/version not found in <REGION>
```

**Fix:** Bootstrap CDK for the account/region combination before the first deploy:
```bash
cdk bootstrap aws://<ACCOUNT_ID>/<REGION>
```

---

### 5. IAM role / CDK chicken-and-egg (two-pass deployment required)

**What happened:** The CDK stack attaches policies to the AgentCore execution role
(passed via `-c agentCoreRoleName=<ROLE_NAME>`). On a fresh account the role doesn't
exist yet. The first deploy failed; its rollback left an orphaned ECR repository.

**Fix (two-pass deployment):**
1. Delete the orphaned ECR repository manually in the AWS console
2. Deploy CDK **without** the role name so the stack can create other resources:
   ```bash
   make deploy-infra AGENTCORE_ROLE_NAME=""
   ```
3. Create the AgentCore execution IAM role manually with the Bedrock trust policy:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Principal": {"Service": "bedrock-agentcore.amazonaws.com"},
       "Action": "sts:AssumeRole"
     }]
   }
   ```
4. Redeploy CDK with the role name to attach the generated policies:
   ```bash
   make deploy-infra
   ```

**CDK gating:** `infrastructure/lib/claude-code-stack.ts` gates the entire policy
attachment block on `if (agentCoreRoleName)`, making step 2 safe.

---

### 6. ECR permissions missing from the AgentCore execution role

**What happened:** After creating the runtime, `create-agent-runtime` failed:
```
ValidationException: Access denied while validating ECR URI
```
The execution role had no ECR read permissions. The CDK stack does not add these
automatically (see item **N** for the fix-needed note).

**Workaround:** Add an inline policy to `<ROLE_NAME>` manually:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "ecr:GetAuthorizationToken",
      "ecr:BatchGetImage",
      "ecr:GetDownloadUrlForLayer"
    ],
    "Resource": "*"
  }]
}
```

---

### 7. Docker image must be ARM64 (`--platform linux/arm64`)

AgentCore runs on Graviton. Building without the platform flag produces an amd64 image
that fails to start silently (no error surfaced to GitHub or CloudWatch).

Always build with:
```bash
docker build --platform linux/arm64 -t <ECR_URI>:latest .
```

---

### 8. Corporate git hooks may block pushes to external GitHub repos

Some corporate environments run git hooks (e.g. a code-scanner hook) that prevent
`git push` to repos not on an allowlist.

**Workaround:** Use the GitHub Contents API to upload workflow files directly:
```bash
gh api repos/<GH_OWNER>/<GH_REPO>/contents/<PATH> \
  -X PUT \
  -f message="chore: add file" \
  -f content="$(base64 -i <LOCAL_FILE>)"
```

---

### 9. `sts:TagSession` required for role assumption in GitHub Actions

**What happened:** The GitHub Actions IAM user could call `sts:AssumeRole` but not
`sts:TagSession`. Configuring `role-to-assume` in `aws-actions/configure-aws-credentials@v4`
requires both permissions.

**Fix:** Add `sts:TagSession` alongside `sts:AssumeRole` in the IAM user's policy.

---

## Part 2: Runtime Issues (Session Behaviour)

Issues that caused agent sessions to fail silently, exit immediately, or produce no code.

---

### 10. Bedrock cross-region inference profile not covered by IAM policy

**Primary cause of first session producing zero code.**

**What happened:** The CDK-generated `AgentCoreBedrockInvokePolicy` allows:
```
arn:aws:bedrock:<REGION>::foundation-model/anthropic.*
```
The default model `us.anthropic.claude-opus-4-6-v1` is a **cross-region inference
profile**. Its ARN prefix is `us.anthropic.*`, not `anthropic.*`. The wildcard
`anthropic.*` does **not** match `us.anthropic.*`.

`InvokeModel` calls silently fail with access denied. Because CloudWatch Logs permissions
were also missing (see #11), no error was ever visible.

**Fix:** Add an inline policy covering inference profiles:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "bedrock:InvokeModel",
      "bedrock:InvokeModelWithResponseStream"
    ],
    "Resource": [
      "arn:aws:bedrock:*::foundation-model/us.anthropic.*",
      "arn:aws:bedrock:*::foundation-model/anthropic.*",
      "arn:aws:bedrock:*:<ACCOUNT_ID>:inference-profile/*",
      "arn:aws:bedrock:*::foundation-model/*"
    ]
  }]
}
```

**Root fix needed:** Update `infrastructure/lib/claude-code-stack.ts` to use
`us.anthropic.*` (or a wildcard that covers both) in the generated policy resource.

---

### 11. CloudWatch Logs permissions missing — agent runs completely blind

**What happened:** The CDK-generated `AgentCoreCloudWatchPolicy` only allows
`cloudwatch:PutMetricData`. No `logs:*` permissions were included. The OTEL exporter
is configured to write to CloudWatch Logs but silently fails to create the log group or
write events.

**Consequence:** Any container error is completely invisible. This made every subsequent
debugging step much harder.

**Fix:** Add inline policy to `<ROLE_NAME>`:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "logs:CreateLogGroup",
      "logs:CreateLogStream",
      "logs:PutLogEvents",
      "logs:DescribeLogGroups",
      "logs:DescribeLogStreams"
    ],
    "Resource": "arn:aws:logs:<REGION>:<ACCOUNT_ID>:log-group:/aws/bedrock-agentcore/*:*"
  }]
}
```

**Root fix needed:** Add `logs:*` to `AgentCoreCloudWatchPolicy` in the CDK stack.

---

### 12. SSM write permissions missing

**What happened:** The CDK-generated policies include no `ssm:PutParameter` permission.
`store_session_state_ssm()` in `bedrock_entrypoint.py` silently fails. The
`/claude-code/session-id` and `/claude-code/current-issue` parameters are never written.

This does not break the agent, but eliminates the health-check signal that the
issue-poller uses to detect stale sessions.

**Fix:** Add inline policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "ssm:PutParameter",
      "ssm:GetParameter",
      "ssm:GetParameters",
      "ssm:DeleteParameter"
    ],
    "Resource": "arn:aws:ssm:<REGION>:<ACCOUNT_ID>:parameter/claude-code/*"
  }]
}
```

---

### 13. `SESSION_DURATION_HOURS` default too short

**What happened:** The Makefile default `SESSION_DURATION_HOURS ?= 1.0` gives the agent
only 1 hour. A typical full-stack build takes 2–6 hours.

**Fix:** Set `SESSION_DURATION_HOURS ?= 7.0` in the Makefile. The Python code already
defaults to `7.0` if the env var is absent; the Makefile was overriding it with a lower
value.

---

### 14. `AWS_REGION` shell leak into `update-runtime-env` breaks Secrets Manager lookups

**Root cause of sessions silently never starting (no GitHub comment, no logs).**

**What happened:** Running `make update-runtime-env` from a shell with
`export AWS_REGION=<WRONG_REGION>` baked that value into the runtime's environment. All
secrets are in `<TARGET_REGION>`. With the wrong region, `get_github_token()` returns
`None`. The handler immediately yields an error and returns — with no GitHub comment, no
SSM write, and no CloudWatch logs (OTEL never ran long enough to export).

**The failure was completely invisible** by every observability channel.

**Fix:** Always pass the region explicitly:
```bash
make update-runtime-env AWS_REGION=<REGION>
```
Or `unset AWS_REGION` first to let the Makefile `?=` default take effect.

**Root fix needed:** In the Makefile, use `CF_REGION` (a non-exported internal variable)
rather than `$(AWS_REGION)` when constructing the runtime env var block:
```makefile
"AWS_REGION": "$(CF_REGION)"
```
This isolates the runtime's AWS_REGION from the shell environment.

---

### 15. OTEL log group name missing `-DEFAULT` suffix (silent log drop)

**What happened:** The Makefile configured OTEL to export logs to:
```
/aws/bedrock-agentcore/runtimes/<RUNTIME_ID>
```
But AgentCore actually creates the log group as:
```
/aws/bedrock-agentcore/runtimes/<RUNTIME_ID>-DEFAULT
```

The ADOT sidecar uses the `x-aws-log-group` header to route logs. Since the name was
wrong, CloudWatch never received any container logs even after fixing the IAM permissions.

**Fix:** Add `-DEFAULT` to both OTEL env vars in the Makefile:
```makefile
OTEL_RESOURCE_ATTRIBUTES = \
  service.name=$(AGENT_NAME),\
  aws.log.group.names=/aws/bedrock-agentcore/runtimes/$(AGENT_RUNTIME_ID)-DEFAULT

OTEL_EXPORTER_OTLP_LOGS_HEADERS = \
  x-aws-log-group=/aws/bedrock-agentcore/runtimes/$(AGENT_RUNTIME_ID)-DEFAULT,\
  x-aws-log-stream=runtime-logs,\
  x-aws-metric-namespace=bedrock-agentcore
```

Apply via `make update-runtime-env`.

---

### 16. Issue poller `check-session-health` job missing role assumption

**What happened:** The `check-session-health` job in `issue-poller.yml` used direct IAM
user credentials (no `role-to-assume`), while the IAM user only had `sts:AssumeRole` and
`sts:TagSession` permissions. The job failed with:
```
AccessDenied: User ... is not authorized to perform: cloudwatch:GetMetricStatistics
```

Every other workflow job correctly assumed a role via `role-to-assume`, but the health
check job was missing this step.

**Fix:** Add `role-to-assume` to the `Configure AWS credentials` step in the
`check-session-health` job:
```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: <REGION>
    role-to-assume: ${{ secrets.AWS_AGENTCORE_ROLE_ARN }}
    role-duration-seconds: 900
```

Also ensure the assumed role has `cloudwatch:GetMetricStatistics` and `ssm:GetParameter`
permissions (see items #11 and #12).

---

### 17. `PROJECT_ROOT` not set — subprocess can't find prompts (immediate exit)

**Root cause of sessions completing in ~7 seconds with no code generated.**

**What happened:** `claude_code.py` uses
`current_dir = os.environ.get('PROJECT_ROOT', os.getcwd())` to locate the
`prompts/<project_name>/` directory. When spawned as a subprocess, `cwd` is the cloned
target repo (e.g. `/app/workspace/agent-runtime`).

If the **target repo does not have `prompts/` committed** (typical for a thin target repo
that only contains `.github/` workflows), `SessionManager.get_project_prompts_dir()`
raises `FileNotFoundError`. `main()` returns immediately (exit code 0). The outer
monitoring loop in `bedrock_entrypoint.py` detects the exited subprocess and calls
`release_github_issue()` → `agent-complete` label — all within ~7 seconds.

In the **original harness repo**, `prompts/` is committed to the repo, so cloning it
gives the subprocess everything it needs. When using a **separate target repo** (the
intended deployment pattern), prompts are baked into the Docker image at `/app/prompts/`
but the subprocess CWD points at the cloned repo which has no prompts.

**Fix applied in `bedrock_entrypoint.py`:** Set `PROJECT_ROOT=/app` in the subprocess
environment so `claude_code.py` always finds prompts from the Docker image:
```python
# In run_agent_background(), in the env setup block:
env['PROJECT_ROOT'] = '/app'
```

This makes the deployment work regardless of whether the target repo contains prompts.

**Root fix needed:** The default in `claude_code.py`
(`os.environ.get('PROJECT_ROOT', os.getcwd())`) should fall back to `/app` rather than
CWD, or `bedrock_entrypoint.py` should always set `PROJECT_ROOT` explicitly.

---

### 18. Stale `agent_state.json` on EFS causes immediate agent-complete on re-run

**What happened:** EFS (`/app/workspace/agent-runtime`) persists across container
restarts. `git pull` does not remove untracked files. When a session completes normally,
`claude_code.py` writes `desired_state: "pause"` to
`generated-app/agent_state.json` on EFS to signal completion. On the next session start
the subprocess finds this stale state, enters `_handle_pause_mode()`, and blocks. The
`bedrock_entrypoint.py` outer loop reads "pause" after its 5-second startup delay and
calls `release_github_issue()` → `agent-complete`. Elapsed time: ~7 seconds.

**First attempted fix:** Called `_set_agent_desired_state("continuous")` before starting
the subprocess. This only updates the file if it already exists — it is a no-op when the
file is absent (e.g. fresh EFS or first ever run).

**Fix applied in `bedrock_entrypoint.py`:** Directly **delete** the stale file rather
than trying to update it. When the file is absent, `claude_code.py` creates it fresh
with `desired_state: "continuous"` on startup:
```python
# Before starting the background thread (only for fresh sessions, not resume):
if not resume_session:
    for stale_path in [
        AGENT_RUNTIME_DIR / "generated-app" / "agent_state.json",
        AGENT_RUNTIME_DIR / "agent_state.json",
    ]:
        if stale_path.exists():
            stale_path.unlink()
            print(f"🧹 Removed stale agent_state.json at {stale_path}")
```

**Note:** `make reset` does not clear EFS — it only deletes the git branch, SSM
parameters, and S3 buckets. The stale state file survives `make reset`.

---

### 19. `invoke_agent_runtime` streaming response parsing error (cosmetic)

The `invoke_agent.py` streaming response parser throws:
```
Warning: Error processing response stream: a bytes-like object is required, not 'str'
```
This is a bug in the boto3 streaming event handler (bytes vs. str mismatch). It does
**not** affect whether the AgentCore session starts — the invocation returns HTTP 200
and the container runs asynchronously. The warning is misleading but harmless.

---

## Part 3: Observability Summary

| Signal | Works? | How to get it |
|--------|--------|--------------|
| CloudWatch Logs | ✅ after fixes | Requires `logs:*` IAM + `-DEFAULT` suffix in OTEL config |
| CloudWatch Metrics (heartbeat) | ✅ | `cloudwatch:PutMetricData` in CDK policy |
| S3 Screenshots | ✅ when agent generates frontend | Set `SCREENSHOT_CDN_DOMAIN` env var |
| SSM Parameters | ✅ after `ssm:PutParameter` added | `/claude-code/session-id`, `/claude-code/current-issue` |
| GitHub issue comments | ✅ | Posted by `bedrock_entrypoint.py` on start and completion |
| `agent-runtime` branch commits | ✅ | Pushed by post-commit hook and periodic push loop |

**Debugging tip:** If a session exits in under 30 seconds with `agent-complete` and no
commits, check in this order:
1. Is `AWS_REGION` correct in the runtime env? (`make get-runtime`)
2. Does the GitHub token exist in Secrets Manager in the correct region?
3. Is `PROJECT_ROOT=/app` set so prompts can be found?
4. Is there a stale `agent_state.json` on EFS? (Delete manually if needed)
5. Check CloudWatch log group `/aws/bedrock-agentcore/runtimes/<RUNTIME_ID>-DEFAULT`

---

## Part 4: Required Code Changes for Open Source / New Forks

These are hardcoded values and structural issues that **must be changed** before this
repo can be used by anyone other than the original author. None require logic changes —
they are all configuration-level fixes.

---

### A. Hardcoded original account number in `bedrock_entrypoint.py`

The fallback value for `AGENT_RUNTIME_ARN` is hardcoded to the original author's
account and runtime ID. This appears in three places:

```python
agent_runtime_arn = os.environ.get(
    "AGENT_RUNTIME_ARN",
    "arn:aws:bedrock-agentcore:<REGION>:<ORIGINAL_ACCOUNT_ID>:runtime/<ORIGINAL_RUNTIME_ID>"
)
```

**Impact:** The "Agent Session Started" GitHub comment shows the wrong Runtime ARN.
Cosmetic only, but confusing.

**Fix:** Add `AGENT_RUNTIME_ARN` as a runtime environment variable, constructed from
`AGENT_RUNTIME_ID` and the AWS account ID. In the Makefile:
```makefile
AGENT_RUNTIME_ARN = arn:aws:bedrock-agentcore:$(CF_REGION):$(AWS_ACCOUNT_ID):runtime/$(AGENT_RUNTIME_ID)
```
Add to `launch` and `update-runtime-env` targets. Remove all hardcoded fallback strings.

---

### B. Hardcoded values in GitHub Actions workflow files

A full audit of all five workflow files. Every item marked **[must fix]** breaks the
workflow for any new deployment.

---

#### `agent-builder.yml`

| Location | Hardcoded value | Severity |
|----------|----------------|----------|
| `env.AWS_REGION` (top-level) | `us-east-1` | **[must fix]** |
| `env.AGENTCORE_AGENT_ID` (top-level) | `<RUNTIME_ID>` | **[must fix]** |
| `invoke_agent.py` call (`--agent-arn` flag) | Full ARN with account number and region: `arn:aws:bedrock-agentcore:us-east-1:<ACCOUNT_ID>:runtime/${AGENTCORE_AGENT_ID}` | **[must fix]** |
| `invoke_agent.py` call (`--region` flag) | `us-east-1` | **[must fix]** |

The account number appears literally in the `--agent-arn` flag even though
`AGENTCORE_AGENT_ID` is already parameterized via the env block above it.

**Fixes:**
- Move `AGENTCORE_AGENT_ID` to a GitHub Actions repository variable
  (`Settings → Variables → Actions`). Reference as `${{ vars.AGENTCORE_AGENT_ID }}`.
- Add `AWS_ACCOUNT_ID` as a repository variable, or derive it at runtime:
  ```bash
  AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
  ```
- Replace the hardcoded ARN with:
  ```bash
  --agent-arn "arn:aws:bedrock-agentcore:${{ vars.AWS_REGION }}:${AWS_ACCOUNT_ID}:runtime/${{ vars.AGENTCORE_AGENT_ID }}"
  ```
- Replace `--region us-east-1` with `--region ${{ vars.AWS_REGION }}` (or `${{ env.AWS_REGION }}`).
- Replace `AWS_REGION: us-east-1` with `AWS_REGION: ${{ vars.AWS_REGION }}`.

---

#### `stop-agent-on-close.yml`

| Location | Hardcoded value | Severity |
|----------|----------------|----------|
| `env.AWS_REGION` (top-level) | `us-east-1` | **[must fix]** |
| Python `AGENT_ARN` string (inline script) | `arn:aws:bedrock-agentcore:us-east-1:<ACCOUNT_ID>:runtime/...` — account number AND region | **[must fix]** |
| `os.environ.get('AGENTCORE_AGENT_ID', ...)` fallback | `<RUNTIME_ID>` hardcoded as default | **[must fix]** |
| `boto3.client(... region_name='us-east-1')` | `us-east-1` | **[must fix]** |

The Python inline script constructs the agent ARN with a literal account number. Even
though `AGENTCORE_AGENT_ID` is passed as an env var, the fallback value is the original
runtime ID. The boto3 client also ignores `env.AWS_REGION` and uses a hardcoded string.

**Fixes:**
- Pass `AWS_ACCOUNT_ID` and `AWS_REGION` as `env:` vars to the Python step.
- Remove the hardcoded fallback from `os.environ.get('AGENTCORE_AGENT_ID', '<RUNTIME_ID>')` — fail loudly if it is missing instead.
- Construct `AGENT_ARN` from env vars:
  ```python
  AGENT_ARN = f"arn:aws:bedrock-agentcore:{os.environ['AWS_REGION']}:{os.environ['AWS_ACCOUNT_ID']}:runtime/{os.environ['AGENTCORE_AGENT_ID']}"
  ```
- Change `boto3.client('bedrock-agentcore', region_name='us-east-1')` to use `os.environ['AWS_REGION']`.

---

#### `deploy-preview.yml`

| Location | Hardcoded value | Severity |
|----------|----------------|----------|
| `env.AWS_REGION` (top-level) | `us-east-1` | **[must fix]** |
| CDK deploy step (`--stack-name`) | `canopy-app-stack` — project-specific stack name | **[must fix]** |

The `--stack-name canopy-app-stack` flag is specific to the Canopy demo project. Any
project using a different CDK stack name will fail to deploy the preview.

S3 bucket names, CloudFront domain, and distribution ID are **correctly** read from
`${{ vars.* }}` repository variables — these are fine.

**Fixes:**
- Replace `AWS_REGION: us-east-1` with `AWS_REGION: ${{ vars.AWS_REGION }}`.
- Add an `APP_CDK_STACK_NAME` repository variable and replace the hardcoded stack name:
  ```yaml
  --stack-name ${{ vars.APP_CDK_STACK_NAME }}
  ```

---

#### `deploy-infrastructure.yml`

| Location | Hardcoded value | Severity |
|----------|----------------|----------|
| `env.AWS_REGION` (top-level) | `us-east-1` | **[must fix]** |

This is the only hardcoded value. All other configuration (bucket names, role ARNs)
appears to use `${{ vars.* }}` or `${{ secrets.* }}` correctly.

**Fix:** Replace `AWS_REGION: us-east-1` with `AWS_REGION: ${{ vars.AWS_REGION }}`.

---

#### `issue-poller.yml`

The `check-session-health` job was missing `role-to-assume` — this was identified and
fixed in the course of deployment (see runtime issue #16). No remaining hardcoded values.

The `poll-and-trigger` job correctly reads `AUTHORIZED_APPROVERS` from
`${{ vars.AUTHORIZED_APPROVERS }}`. Label names (`agent-building`, `agent-complete`,
`tests-failed`) are hardcoded strings but these are intentional constants, not
deployment-specific values.

---

**Summary of required repository variables for all workflows to work:**

| Variable | Used by | Example value |
|----------|---------|--------------|
| `AWS_REGION` | All 5 workflows | `us-east-1` |
| `AGENTCORE_AGENT_ID` | `agent-builder.yml`, `stop-agent-on-close.yml` | `claude_code_reinvent-XXXXXXXXXX` |
| `AWS_ACCOUNT_ID` | `agent-builder.yml`, `stop-agent-on-close.yml` | `123456789012` |
| `APP_CDK_STACK_NAME` | `deploy-preview.yml` | `canopy-app-stack` or `myapp-stack` |
| `AUTHORIZED_APPROVERS` | `issue-poller.yml` | `githubuser1,githubuser2` |
| `SCREENSHOTS_BUCKET_NAME` | `deploy-preview.yml` | (from CDK outputs) |
| `PREVIEWS_BUCKET_NAME` | `deploy-preview.yml` | (from CDK outputs) |
| `PREVIEWS_CDN_DOMAIN` | `deploy-preview.yml` | (from CDK outputs) |
| `PREVIEWS_DISTRIBUTION_ID` | `deploy-preview.yml` | (from CDK outputs) |

Required secrets: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_AGENTCORE_ROLE_ARN`.

---

### C. Hardcoded account-specific values in the Makefile

The Makefile contains defaults that are specific to the original deployment:

```makefile
AGENT_RUNTIME_ID ?= <ORIGINAL_RUNTIME_ID>
EXECUTION_ROLE_ARN ?= arn:aws:iam::<ORIGINAL_ACCOUNT_ID>:role/claude-code-agentcore-role
VPC_ID ?= <ORIGINAL_VPC_ID>
GITHUB_REPO ?= <ORIGINAL_OWNER>/<ORIGINAL_REPO>
```

Any new deployer silently gets the wrong values if they forget to override.

**Fix:**
- Remove account-specific defaults. Replace with empty string or a sentinel like
  `REQUIRED_OVERRIDE` so `make` fails loudly if not set.
- Add a `check-config` target that validates all required variables before running
  sensitive targets (`launch`, `update-runtime-env`, `reset`).
- Example pattern:
  ```makefile
  AGENT_RUNTIME_ID ?= $(error Set AGENT_RUNTIME_ID in your environment or Makefile.local)
  ```

---

### D. `Makefile` `?=` operator is overridden by exported shell variables

`?=` only sets a variable if it is **not already defined** — but `export`ed shell
variables count as defined. So `export AWS_REGION=<WRONG_REGION>` silently overrides
`AWS_REGION ?= <TARGET_REGION>` and all commands run against the wrong region.

Combined with `export AWS_REGION` at the top of the Makefile (which re-exports the
possibly-wrong value into every subprocess), this creates a "sticky" leak that is
extremely hard to diagnose.

**Fix:**
- Remove `export AWS_REGION` from the Makefile top level.
- Use a separate non-exported internal variable for the target region (e.g. `CF_REGION`)
  and pass it explicitly only where needed.
- Document in the README that deployers must `unset AWS_REGION` or override on every
  `make` invocation.

---

### E. Inconsistent model name defaults across files

Four different model references exist in the codebase:

| Location | Value |
|----------|-------|
| `Makefile` `DEFAULT_MODEL` | `us.anthropic.claude-opus-4-6-v1` |
| `bedrock_entrypoint.py` fallback | `us.anthropic.claude-opus-4-6-v1` |
| `src/config.py` constant | `us.anthropic.claude-sonnet-4-5-XXXXXXXXXX-v1:0` (different model, different format) |
| `docker-compose.yml` | `claude-opus-4-5-XXXXXXXX` (non-Bedrock format, no `us.` prefix) |

`src/config.py` uses a **different model** (Sonnet vs Opus) and a **different ID format**
(includes `:0` version suffix). `docker-compose.yml` uses a model ID format that is not
valid for Bedrock cross-region inference profiles.

**Fix:** Standardize to a single source of truth. The `Makefile` `DEFAULT_MODEL` should
be canonical. `src/config.py` and `docker-compose.yml` should be updated to match, or
`src/config.py` should read `os.environ.get("DEFAULT_MODEL")` at runtime.

---

### F. CloudWatch container logs not flowing despite OTEL configuration

Even after fixing the `-DEFAULT` log group suffix (item #15) and IAM permissions
(item #11), Python `logger.info()` calls from inside the container may not appear in
CloudWatch. The `otel-rt-logs` stream is created (confirming the ADOT sidecar
initialized), but remains empty.

**Possible causes:**
1. The container entrypoint does not wrap Python with `opentelemetry-instrument`, so
   Python logging auto-instrumentation is never activated.
2. Log export may require `OTEL_LOGS_EXPORTER=otlp` env var (not set by default).
3. The ADOT sidecar may only forward traces/metrics, not Python logs.

**Diagnostic steps:**
- Check the `Dockerfile` `CMD`/`ENTRYPOINT` — does it include `opentelemetry-instrument`?
- Add `OTEL_LOGS_EXPORTER=otlp` to the runtime env vars and retest.
- Check if `logger.info()` vs `print()` matters (OTEL auto-instrumentation patches
  `logging` module, not `print`).

**Workaround:** The GitHub issue comments (posted via direct API calls from the container)
are the most reliable observability channel until logs are fixed.

---

### G. `GITHUB_REPO` default targets original deployment repo

```makefile
GITHUB_REPO ?= <ORIGINAL_OWNER>/<ORIGINAL_REPO>
```

`make reset` uses this to close issues and delete branches. Running `make reset` without
overriding this on a different deployment would affect the wrong repository.

**Fix:** Remove this default entirely (force explicit override), or auto-detect from:
```bash
gh repo view --json nameWithOwner -q .nameWithOwner
```

---

### H. ECR permissions not included in CDK stack

The CDK stack does not add ECR read permissions to the execution role. Every new
deployment requires a manual inline policy (see item #6).

**Fix needed in `infrastructure/lib/claude-code-stack.ts`:** Add ECR read permissions
alongside the other generated policies:
```typescript
// Within the agentCoreRoleName block:
new iam.Policy(this, 'AgentCoreECRPolicy', {
  statements: [new iam.PolicyStatement({
    effect: iam.Effect.ALLOW,
    actions: [
      'ecr:GetAuthorizationToken',
      'ecr:BatchGetImage',
      'ecr:GetDownloadUrlForLayer',
    ],
    resources: ['*'],
  })],
  roles: [agentCoreRole],
});
```

---

### I. Bedrock inference profile IAM pattern too narrow in CDK stack

The CDK-generated `AgentCoreBedrockInvokePolicy` uses `anthropic.*` which does not
match cross-region inference profiles (`us.anthropic.*`). This must be fixed in the CDK
stack so it works out-of-the-box without manual patching.

**Fix needed in `infrastructure/lib/claude-code-stack.ts`:**
```typescript
new iam.PolicyStatement({
  effect: iam.Effect.ALLOW,
  actions: ['bedrock:InvokeModel', 'bedrock:InvokeModelWithResponseStream'],
  resources: [
    `arn:aws:bedrock:*::foundation-model/us.anthropic.*`,
    `arn:aws:bedrock:*::foundation-model/anthropic.*`,
    `arn:aws:bedrock:*:${this.account}:inference-profile/*`,
    `arn:aws:bedrock:*::foundation-model/*`,
  ],
})
```

---

### J. `PROJECT_ROOT` not set in subprocess environment

Documented under runtime issue #17. The code fix (`env['PROJECT_ROOT'] = '/app'`) has
been applied to `bedrock_entrypoint.py`. The root fix needed is to make this the
default behavior in `claude_code.py` rather than requiring `bedrock_entrypoint.py` to
set it explicitly.

---

### K. Stale `agent_state.json` on EFS survives `make reset`

Documented under runtime issue #18. The code fix (delete state files before subprocess
start) has been applied to `bedrock_entrypoint.py`. Additionally, `make reset` should
optionally delete the EFS `generated-app/` directory to give a truly clean slate:

```makefile
reset-efs:
    # Run inside a container that mounts the EFS volume
    aws ecs run-task ... --command "rm -rf /app/workspace/agent-runtime/generated-app"
```

Or document that `make reset` alone is not sufficient for a fully clean EFS state.

---

## Part 5: Deployment Checklist for New Forks

Use this checklist when deploying to a new AWS account or GitHub repository.

### Before deploying infrastructure
- [ ] `unset AWS_REGION` (or verify it is set to your target region)
- [ ] Delete `infrastructure/package-lock.json` and regenerate with `npm install`
- [ ] Run `cdk bootstrap aws://<ACCOUNT_ID>/<REGION>`
- [ ] Set `AGENT_RUNTIME_ID`, `EXECUTION_ROLE_ARN`, `VPC_ID`, `GITHUB_REPO` in Makefile
      or a `Makefile.local` file

### Infrastructure deployment
- [ ] First pass: `make deploy-infra AGENTCORE_ROLE_NAME=""`
- [ ] Create the AgentCore execution role manually with Bedrock trust policy
- [ ] Second pass: `make deploy-infra`
- [ ] Add ECR permissions inline policy to execution role (see item #6)
- [ ] Add Bedrock inference profile permissions inline policy (see item #10)
- [ ] Add CloudWatch Logs permissions inline policy (see item #11)
- [ ] Add SSM permissions if not already present (see item #12)

### Building and pushing the Docker image
- [ ] `aws ecr get-login-password --region <REGION> | docker login ...`
- [ ] `docker build --platform linux/arm64 -t <ECR_URI>:latest .`
- [ ] `docker push <ECR_URI>:latest`

### Runtime creation and configuration
- [ ] `make update-runtime-env AWS_REGION=<REGION>` (always pass region explicitly)
- [ ] `make get-runtime` — wait for `"status": "READY"`

### GitHub repository setup
- [ ] Create target repo (e.g. `<GH_OWNER>/<GH_REPO>`)
- [ ] Upload all `.github/workflows/*.yml` files
- [ ] Set repository secrets: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`,
      `AWS_AGENTCORE_ROLE_ARN`, `GITHUB_TOKEN` (usually auto-provided)
- [ ] Set repository variables (see item B for full list):
      - `AWS_REGION` — your target region (used by all 5 workflows)
      - `AGENTCORE_AGENT_ID` — your runtime ID
      - `AWS_ACCOUNT_ID` — your 12-digit AWS account number
      - `APP_CDK_STACK_NAME` — the CDK stack name for the generated app
      - `AUTHORIZED_APPROVERS` — comma-separated GitHub usernames for rocket-reaction approval
      - `SCREENSHOTS_BUCKET_NAME`, `PREVIEWS_BUCKET_NAME`, `PREVIEWS_CDN_DOMAIN`,
        `PREVIEWS_DISTRIBUTION_ID` — from CDK stack outputs
- [ ] Create required labels: `agent-building` (yellow), `agent-complete` (green),
      `tests-failed` (red)

### First run
- [ ] `make reset`
- [ ] Create an issue on the target repo
- [ ] Add 🚀 reaction (from an `AUTHORIZED_APPROVERS` account)
- [ ] `gh workflow run "Agent Builder" --repo <GH_OWNER>/<GH_REPO> -f issue_number=<N>`
- [ ] Watch for `agent-building` label to persist beyond 30 seconds
      (if it flips to `agent-complete` within 7 seconds, see Part 2 debugging tip)
