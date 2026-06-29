# `claude-audit.yml` — Scheduled Repository Audit (pluggable audit types)

Runs an automated, read-only audit on a schedule (or on demand), writes a dated Markdown
report, and opens a PR for review. The workflow is **audit-type-agnostic**: it loads a chosen
plugin's skills from `gemma-agentic-toolkit`, runs that plugin's **orchestrator skill**, and
publishes the result. Two audit types ship today:

| Audit type | `plugin` | `orchestrator_skill` | Audits |
|---|---|---|---|
| Deployment security (default) | `gemma-deployment-security` | `generate-security-report` | Docker Compose, Dockerfile, CI/CD, Airflow config, IaC, pre-commit |
| dbt repository | `gemma-dbt` | `validate-repo` | Gemma SQL style, naming, logic, target-structure, CLAUDE.md, Kimball |

Client repos are usually split (the Airflow *deployment* repo vs the *dbt* repo), so each repo
adds the one caller relevant to it. One run = one audit type = one report = one PR.

---

## Architecture

```
Client repo                    Gemma-Analytics/.github              gemma-agentic-toolkit     AWS
─────────────                  ────────────────────────             ─────────────────────     ───
.github/workflows/
  <audit>.yml
    │  on: schedule             claude-audit.yml (reusable)
    │  on: workflow_dispatch    ┌──────────────────────────────────────────────────────────────────┐
    └──── workflow_call ───────▶│ 1. Checkout client repo                                          │
                                │ 2. App token (client-org) + 3. App token (Gemma-Analytics) ────▶ Gemma Claude App
                                │ 4. Checkout gemma-agentic-toolkit @ toolkit_ref ──────────────▶  plugins/<plugin>/
                                │ 5. OIDC → assume IAM role ─────────────────────────────────────▶ AWS Bedrock
                                │ 6. Copy plugins/<plugin>/skills/. → .claude/skills/              │
                                │ 7. Build generic prompt:                                         │
                                │      read .claude/skills/<orchestrator_skill>/SKILL.md, follow it │
                                │      write report to <report_dir>/<slug>-audit-report-DATE.md     │
                                │      + extra_instructions                                        │
                                │ 8. claude-code-action@v1  (tools = allowed_tools) ─────────────▶ Claude (Bedrock)
                                │ 9. Safety net: move report if written to repo root               │
                                │ 10. Clean up: rm -rf .gemma-toolkit .claude                      │
                                │ 11. peter-evans/create-pull-request@v7                           │
                                │      branch audit/<slug>-DATE · title "<pr_title_prefix> DATE"    │
                                │      body = the report's Executive-summary section               │
                                │ 12. Post session cost to PR                                      │
                                └──────────────────────────────────────────────────────────────────┘
```

---

## The audit-orchestrator contract

Any plugin can be an audit type if its orchestrator skill satisfies this contract — that's
what lets one workflow drive all of them:

1. **`disable-model-invocation: true`**, `argument-hint: "[path-to-<target>]"` (where `<target>`
   names what the skill audits — e.g. `repo-root`, `dbt-project`, `airflow-dags`), and it accepts
   `$ARGUMENTS` as that target directory (defaults to the cwd).
2. **Auto-detects** which sub-checks apply by scanning the repo; records skipped ones with a reason.
3. **Read-only** — never modifies the audited repo; never runs anything that returns warehouse
   data (e.g. `dbt show`) or reads secrets.
4. Writes a dated Markdown report containing an **`## Executive summary`** section (matched
   case-insensitively) — the workflow extracts that block for the PR body.

Both `generate-security-report` and `validate-repo` already satisfy this. To add a new audit
type, ship a plugin whose orchestrator meets the contract, then add a thin caller (below) — no
change to this workflow.

---

## Client wrappers

### Deployment-security audit (the default)

`.github/workflows/scheduled-audit.yml` in the **deployment** repo. With defaults reproducing
the security audit, the `with:` block can be omitted entirely:

```yaml
name: Scheduled Security Audit
on:
  schedule: [{ cron: "0 6 1 * *" }]   # 06:00 UTC, 1st of each month
  workflow_dispatch:
permissions: { id-token: write, contents: write, pull-requests: write }
jobs:
  audit:
    uses: Gemma-Analytics/.github/.github/workflows/claude-audit.yml@<SHA>
    # No `with:` needed — defaults are the deployment-security audit.
    # If your repo has no SSH-based server audit in scope, you can defer it explicitly:
    with:
      extra_instructions: "Skip audit-server-config in this CI run; record it as Deferred."
    secrets:
      CLAUDE_CODE_ROLE_ARN: ${{ secrets.CLAUDE_CODE_ROLE_ARN }}
      GEMMA_CLAUDE_ASSISTANT_APP_ID: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_ID }}
      GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY }}
```

### dbt repository audit

`.github/workflows/dbt-audit.yml` in the **dbt** repo. `validate-repo` fans out parallel
sub-agents, so its caller must allow the `Task` tool:

```yaml
name: dbt Repo Audit
on:
  schedule: [{ cron: "0 7 1 * *" }]   # offset from the security audit
  workflow_dispatch:
permissions: { id-token: write, contents: write, pull-requests: write }
jobs:
  audit:
    uses: Gemma-Analytics/.github/.github/workflows/claude-audit.yml@<SHA>
    with:
      plugin: gemma-dbt
      orchestrator_skill: validate-repo
      report_slug: dbt
      pr_title_prefix: "📊 dbt audit"
      allowed_tools: "Task,Bash(find *),Bash(grep *),Bash(cat *),Bash(ls *),Bash(dbt parse),Read,Glob,Grep,Write"
      extra_instructions: |
        Automated, unattended run. Do not ask any questions; at validate-repo's Phase C5 do NOT
        ask whether to save — write the consolidated report to the path above and continue.
        dbt is not installed here: let `dbt parse` fail and proceed in filename-only mode
        (note the degradation in the report).
    secrets:
      CLAUDE_CODE_ROLE_ARN: ${{ secrets.CLAUDE_CODE_ROLE_ARN }}
      GEMMA_CLAUDE_ASSISTANT_APP_ID: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_ID }}
      GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY }}
```

> **dbt CI fidelity:** the runner has no dbt/profile, so `dbt parse` fails and `validate-repo`
> runs in filename-only mode — only the two manifest-dependent logic checks (dead-model,
> circular-dependency) are skipped; all other dimensions run fully. The report notes this.

> **SHA pinning:** pin `@<SHA>` (not `@main`) for unattended runs. Get it with
> `git ls-remote https://github.com/Gemma-Analytics/.github.git refs/heads/main`.
> **`secrets: inherit` does not work cross-org** — pass all three explicitly.

---

## Inputs

| Input | Default | Description |
|---|---|---|
| `plugin` | `gemma-deployment-security` | Plugin under `.gemma-toolkit/plugins/` whose `skills/` are loaded. |
| `orchestrator_skill` | `generate-security-report` | Orchestrator skill to run (must meet the contract above). |
| `report_slug` | `security` | Names the report (`<slug>-audit-report-DATE.md`) and branch (`audit/<slug>-DATE`). |
| `pr_title_prefix` | `🔒 Security audit` | PR title; the run date is appended. |
| `allowed_tools` | read-only set¹ | `claude-code-action --allowedTools`. Add `Task` for audits that fan out sub-agents. |
| `extra_instructions` | _(empty)_ | Appended to the prompt — audit-type CI notes (e.g. defer a sub-check, suppress an interactive step). |
| `model` | `eu.anthropic.claude-sonnet-4-6` | Bedrock model ID. Sonnet is the default for both audits. |
| `aws_region` | `eu-central-1` | AWS region for Bedrock OIDC. |
| `max_turns` | `60` | Maximum agentic turns. |
| `report_dir` | `docs/audit-reports` | Directory for the dated report. |
| `toolkit_ref` | `main` | `gemma-agentic-toolkit` ref for the skills. Pin to a release tag for reproducibility. |

¹ `Bash(find *),Bash(grep *),Bash(cat *),Bash(ls *),Read,Glob,Grep,Write`

## Secrets (all required)

| Secret | Purpose |
|---|---|
| `CLAUDE_CODE_ROLE_ARN` | IAM role ARN for OIDC → Bedrock |
| `GEMMA_CLAUDE_ASSISTANT_APP_ID` | GitHub App ID (bot identity + cross-org toolkit checkout) |
| `GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY` | GitHub App private key (PEM) |

No new secrets or IAM beyond what `claude-review.yml` already uses — scheduled runs emit the
same OIDC `sub` claim, covered by every `allow_all_repos` trust policy.

---

## Adding a new audit type

1. Ship a plugin in `gemma-agentic-toolkit` whose orchestrator skill meets the contract above.
2. Add a thin caller in the relevant repo setting `plugin` / `orchestrator_skill` / `report_slug`
   / `pr_title_prefix` (and `allowed_tools` if it needs sub-agents). Nothing in this workflow
   changes.
