# Gemma Analytics — Shared GitHub Workflows

Reusable GitHub Actions workflows for all Gemma Analytics repositories. Add thin wrapper files to any repo to get Claude-powered code review, an `@claude` mention responder, and scheduled security audits.

---

## Workflows

| Workflow | What it does | Docs |
|---|---|---|
| [`claude-review.yml`](.github/workflows/claude-review.yml) | Automated code review when a PR is opened or marked ready for review; can also be triggered on demand by commenting `@claude review` on a PR | [docs/claude-review.md](docs/claude-review.md) |
| [`claude.yml`](.github/workflows/claude.yml) | General-purpose `@claude` mention handler — answer questions, help with issues, respond to inline review comments | [docs/claude.md](docs/claude.md) |
| [`claude-audit.yml`](.github/workflows/claude-audit.yml) | Scheduled repository audit (pluggable type — deployment-security or dbt) — runs monthly (or on demand), writes a dated Markdown report, and opens a PR for review | [docs/claude-audit.md](docs/claude-audit.md) |

All workflows authenticate to Claude via **AWS Bedrock** (OIDC, no API key stored). GitHub interactions use a **GitHub App token** (Gemma Claude Assistant) so comments and reactions appear under the bot account rather than a personal token.

---

## Quick start — adding workflows to a new repo

Create the relevant files in your repo's `.github/workflows/` directory.

### 1. Auto-review on PR open (`claude-review.yml`)

```yaml
name: Claude Code Review

on:
  pull_request:
    types: [opened, ready_for_review]
  issue_comment:
    types: [created]

permissions:
  id-token: write
  contents: read
  pull-requests: write
  issues: write

jobs:
  review:
    if: |
      github.event_name == 'pull_request' ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       startsWith(github.event.comment.body, '@claude review'))
    uses: Gemma-Analytics/.github/.github/workflows/claude-review.yml@main
    with:
      additional_instructions: |
        - Any repo-specific rules Claude should check (one per line)
        - e.g. Foreign keys use on_delete=PROTECT
        - e.g. No secrets committed
    secrets: inherit
```

### 2. `@claude` mention responder (`claude.yml`)

```yaml
name: Claude Code

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]

permissions:
  id-token: write
  contents: write
  pull-requests: write
  issues: write

jobs:
  claude:
    if: |
      (github.event_name == 'issue_comment' &&
       contains(github.event.comment.body, '@claude') &&
       !startsWith(github.event.comment.body, '@claude review')) ||
      (github.event_name == 'pull_request_review_comment' &&
       contains(github.event.comment.body, '@claude') &&
       !startsWith(github.event.comment.body, '@claude review')) ||
      (github.event_name == 'issues' && contains(github.event.issue.body, '@claude'))
    uses: Gemma-Analytics/.github/.github/workflows/claude.yml@main
    secrets: inherit
```

> **Important:** the negation `!startsWith(..., '@claude review')` in `claude.yml` ensures that an `@claude review` comment routes to the review workflow, not the generic responder. Keep both files in sync.

### 3. Scheduled repository audit (`claude-audit.yml`)

Creates a dated Markdown report under `docs/audit-reports/` and opens a PR for it — monthly by default, or any time via manual dispatch. The workflow is **audit-type-agnostic**: it loads a plugin's skills and runs that plugin's orchestrator. Two types ship today — **deployment-security** (`gemma-deployment-security` / `generate-security-report`) and **dbt** (`gemma-dbt` / `validate-repo`). See [docs/claude-audit.md](docs/claude-audit.md) for the dbt caller and the orchestrator contract.

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
    uses: Gemma-Analytics/.github/.github/workflows/claude-audit.yml@main
    # Defaults run the deployment-security audit — no `with:` needed.
    secrets:
      CLAUDE_CODE_ROLE_ARN: ${{ secrets.CLAUDE_CODE_ROLE_ARN }}
      GEMMA_CLAUDE_ASSISTANT_APP_ID: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_ID }}
      GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY }}
```

Commit this as `.github/workflows/scheduled-audit.yml`. The workflow needs `contents: write` and `pull-requests: write` (in addition to `id-token: write` for OIDC) because it opens a PR with the report. No new secrets or IAM changes are required — the same three secrets already configured for PR reviews cover the audit too.

> **Note:** `secrets: inherit` does not work cross-org. All three secrets must be passed explicitly, as shown above.

---

## Required secrets

All three secrets must be available in the repo (or inherited from the org). They are already configured at the org level for all Gemma Analytics repositories — no per-repo action is needed.

| Secret | Purpose |
|---|---|
| `CLAUDE_CODE_ROLE_ARN` | AWS IAM role ARN for OIDC; grants access to Bedrock |
| `GEMMA_CLAUDE_ASSISTANT_APP_ID` | GitHub App ID for posting comments and reactions as the bot |
| `GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY` | GitHub App private key (PEM) |

---

## Configuration options

### `claude-review.yml` inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `model` | string | `eu.anthropic.claude-sonnet-4-6` | Bedrock model ID. Change to use a different Claude model (e.g. `eu.anthropic.claude-opus-4-7` for harder reviews). |
| `aws_region` | string | `eu-central-1` | AWS region for Bedrock OIDC. All Gemma infra is in `eu-central-1`; only change if a repo's workload is in another region. |
| `additional_instructions` | string | _(empty)_ | Repo-specific review rules appended to the base prompt. Use one bullet per line. Claude checks these on top of the standard review criteria. |
| `max_turns` | string | `50` | Maximum agentic turns. Caps cost on large diffs; raised to 50 to support exhaustive category sweeps. |

### `claude.yml` inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `model` | string | `eu.anthropic.claude-sonnet-4-6` | Bedrock model ID. |
| `aws_region` | string | `eu-central-1` | AWS region for Bedrock OIDC. |
| `prompt` | string | _(empty)_ | When empty, the action reads the triggering comment/issue body and handles `@claude` mentions natively. Pass an explicit prompt only for special routing (rarely needed). |
| `max_turns` | string | _(empty — action default)_ | When blank, the action uses its built-in default. |

### `claude-audit.yml` inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `plugin` | string | `gemma-deployment-security` | Toolkit plugin whose `skills/` are loaded (e.g. `gemma-dbt` for the dbt audit). |
| `orchestrator_skill` | string | `generate-security-report` | Orchestrator skill to run (e.g. `validate-repo`). Must meet the audit-orchestrator contract — see [docs](docs/claude-audit.md). |
| `report_slug` | string | `security` | Names the report (`<slug>-audit-report-DATE.md`) and branch (`audit/<slug>-DATE`). |
| `pr_title_prefix` | string | `🔒 Security audit` | PR title; the run date is appended. |
| `allowed_tools` | string | read-only set | `claude-code-action --allowedTools`. Add `Task` for audits that fan out sub-agents (e.g. the dbt audit). |
| `extra_instructions` | string | _(empty)_ | Audit-type CI notes appended to the prompt. |
| `model` | string | `eu.anthropic.claude-sonnet-4-6` | Bedrock model ID. |
| `aws_region` | string | `eu-central-1` | AWS region for Bedrock OIDC. |
| `max_turns` | string | `60` | Maximum agentic turns. |
| `report_dir` | string | `docs/audit-reports` | Directory for the dated report (`<report_dir>/<slug>-audit-report-YYYY-MM-DD.md`). |
| `toolkit_ref` | string | `main` | `gemma-agentic-toolkit` branch, tag, or SHA. Pin to a release tag for reproducible audits. |

---

## How triggers work

### Review philosophy

The review is one-shot: auto-fires on PR open and draft→ready; new commits do NOT retrigger it. Re-review requires an explicit `@claude review` comment. The prompt enforces a 9-category sweep with a mandatory completeness self-check, so all blocking issues are surfaced in the first pass rather than drip-fed across rounds. If you observe repeated drip-feed rounds across multiple PRs, treat that as a prompt regression and flag it.

### Automatic review (on PR events)

| Event | Fires? | Notes |
|---|---|---|
| PR opened | Yes | First review on every new PR |
| PR marked ready for review | Yes | Draft → ready transition; ensures draft PRs also get reviewed when they're ready |
| New commit pushed to PR | **No** | Dropped to avoid re-running on every rebase |
| PR reopened | No | Not wired; add `reopened` to `pull_request.types` in your consumer wrapper if needed |

### On-demand review (via comment)

Post `@claude review` as a **PR conversation comment** (the main timeline, not inline on a diff line) to trigger a re-review at any time.

- `@claude review` — re-runs the full review
- `@claude review focus on the auth changes` — re-reviews with the extra text appended as focus guidance in the prompt
- The bot reacts with 👀 to acknowledge within a few seconds
- Concurrency: if you trigger two reviews in quick succession, the first run is cancelled and only the second proceeds

### `@claude` generic mentions

Any comment containing `@claude` (but **not** starting with `@claude review`) routes to the general-purpose responder. Examples:

| Comment | Routes to |
|---|---|
| `@claude review` | Review workflow |
| `@claude review focus on security` | Review workflow |
| `@claude what does this function do?` | Generic responder |
| `@claude please help me fix this` | Generic responder |
| `@claude please review this` | Generic responder (does not start with `@claude review`) |

> **Tip:** `@claude review` is case-sensitive. `@Claude review` (capital C) will not trigger the review workflow — it will route to the generic responder.

---

## Pinning to a specific version

All consumers in this org pin to `@main`, meaning any merge to `main` here is immediately live across all repos. This is intentional for fast iteration.

If you need a stable pin (e.g. for a production-critical repo), reference a tag instead:

```yaml
uses: Gemma-Analytics/.github/.github/workflows/claude-review.yml@v1
```

Tags are not currently cut automatically — coordinate with the infrastructure team.

---

## Security notes

- **Comment body injection:** the `github.event.comment.body` value (user-supplied) is handled via shell `env:` variables, never interpolated directly into `run:` scripts. This prevents command injection from maliciously crafted comments.
- **Fork PRs:** OIDC credentials and secrets are not available to workflows triggered by PRs from forks. Review will silently fail for fork PRs — this is a known GitHub limitation.
- **Bot loops:** the `allowed_bots` setting in both central workflows permits only `gemma-claude-assistant[bot]` and `gemmbot`. Claude's own review summary comments will not re-trigger another review.
- **Audit tool scope:** `claude-audit.yml` restricts Claude to read-only tools (`find`, `grep`, `cat`, `ls`, `Read`, `Glob`, `Grep`) plus a single `Write` for the report file. No `gh` CLI, no network calls, no git commands — PR creation is handled deterministically by `peter-evans/create-pull-request`, not by Claude.
- **Scheduled runs and OIDC:** `schedule` and `workflow_dispatch` runs on the default branch emit an OIDC `sub` claim of `repo:<org>/<repo>:ref:refs/heads/main`. All current `allow_all_repos` clients have trust policies scoped to `repo:<org>/*`, which matches — no Terraform changes are required.
