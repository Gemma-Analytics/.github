# `claude-audit.yml` — Scheduled Security Audit

Runs an automated security audit on the first of each month (or manually via workflow dispatch). Uses the `gemma-deployment-security` plugin from `gemma-agentic-toolkit` to auto-detect and audit applicable layers (Docker Compose, Dockerfile, CI/CD workflows, Airflow config, infrastructure-as-code, pre-commit). Writes a dated Markdown report and opens a PR for review.

---

## Architecture

```
Client repo                    Gemma-Analytics/.github              gemma-agentic-toolkit     AWS
─────────────                  ────────────────────────             ─────────────────────     ───
.github/workflows/
  scheduled-audit.yml
    │
    │  on: schedule             claude-audit.yml (reusable)
    │  (1st of month)           ┌──────────────────────────────────────────────────────────────────┐
    │  on: workflow_dispatch    │ 1. Checkout client repo                                          │
    └──── workflow_call ───────▶│                                                                  │
                                │ 2. App token (client-org-scoped) ──────────────────────────────▶ Gemma Claude
                                │                                                                  │ Assistant App
                                │ 3. App token (Gemma-Analytics-scoped) ────────────────────────▶ │
                                │                                                                  │
                                │ 4. Checkout gemma-agentic-toolkit ───────────────────────────▶  plugins/
                                │    ref: toolkit_ref (default: main)          gemma-deployment-   │
                                │    token: Gemma-Analytics-scoped             security/skills/    │
                                │                                                                  │
                                │ 5. OIDC → assume IAM role ──────────────────────────────────────▶ AWS Bedrock
                                │    (CLAUDE_CODE_ROLE_ARN)                                        │ (eu-central-1)
                                │                                                                  │
                                │ 6. Copy skills → .claude/skills/                                 │
                                │    mkdir -p docs/audit-reports/                                  │
                                │                                                                  │
                                │ 7. Build prompt                                                  │
                                │    └─ read .claude/skills/generate-security-report/SKILL.md      │
                                │    └─ write report to docs/audit-reports/YYYY-MM-DD.md           │
                                │                                                                  │
                                │ 8. claude-code-action@v1 ──────────────────────────────────────▶ Claude
                                │    tools: Read, Grep, Glob, Bash(find/grep/cat/ls), Write        │ (Bedrock)
                                │    └─ auto-detects applicable audit skills                       │
                                │    └─ runs each skill                                            │
                                │    └─ writes dated report file                                   │
                                │                                                                  │
                                │ 9. Safety net: move report if written to repo root               │
                                │                                                                  │
                                │ 10. Clean up: rm -rf .gemma-toolkit .claude                      │
                                │                                                                  │
                                │ 11. peter-evans/create-pull-request@v7                           │
                                │     branch: audit/security-YYYY-MM-DD                           │
                                │     title:  🔒 Security audit YYYY-MM-DD                         │
                                │     body:   executive summary from report                        │
                                │     adds:   docs/audit-reports/ only                             │
                                │                                                                  │
                                │ 12. Post session cost to PR                                      │
                                └──────────────────────────────────────────────────────────────────┘
```

### Auto-detection table

The `generate-security-report` skill detects which sub-audits apply based on files present in the repo:

| Skill | Triggered when |
|---|---|
| `audit-docker-compose` | `docker-compose.yaml` or `docker-compose.yml` found |
| `audit-dockerfile` | `Dockerfile` (any name) found |
| `audit-cicd-workflows` | `.github/workflows/` directory contains YAML files |
| `audit-infrastructure-as-code` | `.tf` files or `ansible/` directory found |
| `audit-airflow-config` | `airflow.cfg` present, or Dockerfile references `apache/airflow` |
| `audit-pre-commit` | `.pre-commit-config.yaml` found |
| `audit-server-config` | **Always deferred** in CI — requires SSH access; run manually |

### Report lifecycle

```
claude-code-action runs
        │
        ▼
docs/audit-reports/security-audit-report-YYYY-MM-DD.md  (written by Claude)
        │
        ▼
peter-evans/create-pull-request
        │
        ├─▶  branch: audit/security-YYYY-MM-DD
        │
        └─▶  PR: 🔒 Security audit YYYY-MM-DD
                  body: executive summary (severity table + overall assessment)
                  full report linked in body footer
```

---

## Client wrapper

Create `.github/workflows/scheduled-audit.yml` in the client repo:

```yaml
name: Scheduled Security Audit

on:
  schedule:
    - cron: "0 6 1 * *"   # 06:00 UTC on the 1st of every month
  workflow_dispatch:        # also triggerable manually from the Actions tab

permissions:
  id-token: write
  contents: write
  pull-requests: write

jobs:
  audit:
    uses: Gemma-Analytics/.github/.github/workflows/claude-audit.yml@<SHA>
    with:
      audits: "deployment-security"
    secrets:
      CLAUDE_CODE_ROLE_ARN: ${{ secrets.CLAUDE_CODE_ROLE_ARN }}
      GEMMA_CLAUDE_ASSISTANT_APP_ID: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_ID }}
      GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY }}
```

> **SHA pinning:** use a specific commit SHA rather than `@main`. Because this workflow runs unattended on a schedule, pinning prevents a central change from affecting your repo without a deliberate version bump. Get the current SHA with:
> ```
> git ls-remote https://github.com/Gemma-Analytics/.github.git refs/heads/main
> ```

> **`secrets: inherit` does not work cross-org.** All three secrets must be passed explicitly, as shown.

---

## Inputs

| Input | Default | Description |
|---|---|---|
| `audits` | `deployment-security` | Audit types to run (space or newline separated). Currently supported: `deployment-security`. Future audit types (e.g. `dbt-project-review`) will use the same input — add to the list when they ship. |
| `model` | `eu.anthropic.claude-sonnet-4-6` | Bedrock model ID. The audit is read-heavy (~30–50 turns); Sonnet is the right trade-off. Switch to Opus only if detection quality is inadequate. |
| `aws_region` | `eu-central-1` | AWS region for Bedrock OIDC. |
| `max_turns` | `60` | Maximum agentic turns. A full deployment-security run takes ~30–50 turns; 60 gives headroom for larger repos. Raise if the run is being cut short on complex repos. |
| `report_dir` | `docs/audit-reports` | Directory where dated reports are written. The full path is `<report_dir>/security-audit-report-YYYY-MM-DD.md`. Change if your repo uses a different docs layout. |
| `toolkit_ref` | `main` | `gemma-agentic-toolkit` branch, tag, or SHA. Pin to a release tag (e.g. `gemma-deployment-security@1.1.0`) for fully reproducible audits where skill updates don't silently change findings. |

## Secrets (all required)

| Secret | Purpose |
|---|---|
| `CLAUDE_CODE_ROLE_ARN` | IAM role ARN for OIDC; grants Bedrock invoke access |
| `GEMMA_CLAUDE_ASSISTANT_APP_ID` | GitHub App ID for bot identity |
| `GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY` | GitHub App private key (PEM) |

No new secrets or IAM changes are needed beyond what's already configured for `claude-review.yml`. The same three secrets and the same IAM role cover the audit too (scheduled runs emit the same OIDC `sub` claim as branch-triggered runs).

---

## Customising audit scope

**Change the schedule:** edit the `cron` expression in the client wrapper.

**Run only specific sub-audits:** currently the `generate-security-report` skill auto-detects and runs all applicable audits. Finer-grained control per sub-skill is not yet exposed as an input.

**Pin the skills version:** set `toolkit_ref` to a tag or SHA in `gemma-agentic-toolkit` so a skills update doesn't change your findings unexpectedly:

```yaml
with:
  toolkit_ref: "gemma-deployment-security@1.1.0"
```

**Change the report directory:**

```yaml
with:
  report_dir: "security/reports"
```

**Use a more capable model for a one-off deep audit:**

```yaml
with:
  model: "eu.anthropic.claude-opus-4-8"
  max_turns: "80"
```

---

## OIDC trust note

`schedule` and `workflow_dispatch` runs on the default branch emit an OIDC `sub` claim of `repo:<org>/<repo>:ref:refs/heads/main`. All clients with `allow_all_repos = true` in their Terraform trust policy already cover this claim — no infrastructure changes are needed to enable scheduled audits on existing clients.
