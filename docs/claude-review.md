# `claude-review.yml` — Automated PR Code Review

Runs a structured, one-shot code review whenever a pull request is opened or moved out of draft. Can also be re-triggered on demand via `@claude review` in any PR comment.

---

## Architecture

```
Client repo                       Gemma-Analytics/.github              AWS / GitHub
─────────────                     ────────────────────────             ─────────────
.github/workflows/
  claude-review.yml
    │
    │  on: pull_request            claude-review.yml (reusable)
    │  on: issue_comment           ┌─────────────────────────────────────────────────┐
    │  (@claude review)            │ 1. Checkout client repo                         │
    └──── workflow_call ──────────▶│                                                 │
                                   │ 2. Create GitHub App token ────────────────────▶│ Gemma Claude
                                   │    (Gemma Claude Assistant)                     │ Assistant App
                                   │                                                 │
                                   │ 3. OIDC → assume IAM role ─────────────────────▶│ AWS Bedrock
                                   │    (CLAUDE_CODE_ROLE_ARN)                       │ (eu-central-1)
                                   │                                                 │
                                   │ 4. React 👀 to trigger comment                  │
                                   │    (if issue_comment event)                     │
                                   │                                                 │
                                   │ 5. Build prompt                                 │
                                   │    └─ 9-category sweep instructions             │
                                   │    └─ append additional_instructions            │
                                   │    └─ append user focus text (from comment)     │
                                   │                                                 │
                                   │ 6. claude-code-action@v1 ──────────────────────▶│ Claude
                                   │    ├─ reads CLAUDE.md (if present)              │ (Bedrock)
                                   │    ├─ fetches PR diff via gh pr diff            │
                                   │    ├─ posts inline comments (mcp_inline)        │
                                   │    └─ posts summary via gh pr comment           │
                                   │                                                 │
                                   │ 7. Post session cost comment                    │
                                   └─────────────────────────────────────────────────┘
```

### Trigger routing

```
PR opened ──────────────────────────────────────────────────────────▶ review fires
PR draft → ready ───────────────────────────────────────────────────▶ review fires
New commit pushed ───────────────────────────────────────────────────▶ (no-op)

Comment: "@claude review" ───────────────────────────────────────────▶ review fires
Comment: "@claude review focus on auth" ─────────────────────────────▶ review fires (with focus)
Comment: "@claude what does this do?" ───────────────────────────────▶ routes to claude.yml
```

### Concurrency

One review per PR at a time. If two reviews are triggered in quick succession (e.g. two rapid `@claude review` comments), the first run is **cancelled** and only the second proceeds.

---

## Client wrapper

Create `.github/workflows/claude-review.yml` in the client repo:

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
        - Any repo-specific rules (one per line)
        - e.g. Foreign keys must use on_delete=PROTECT
        - e.g. No dbt models without a schema.yml entry
    secrets: inherit
```

---

## Inputs

| Input | Default | Description |
|---|---|---|
| `model` | `eu.anthropic.claude-sonnet-4-6` | Bedrock model ID. Upgrade to `eu.anthropic.claude-opus-4-8` for harder reviews at higher cost. |
| `aws_region` | `eu-central-1` | AWS region for Bedrock OIDC. All Gemma infra is in `eu-central-1`; only change if the repo's workload is in another region. |
| `additional_instructions` | _(empty)_ | Repo-specific rules appended to the base 9-category prompt. One bullet per line. Claude checks these on top of the standard sweep. Use this for domain-specific contracts (schema conventions, required test patterns, etc.). |
| `max_turns` | `50` | Maximum agentic turns. Raised from the action default to support the 9-category sweep on large diffs. Reduce if costs are too high on a specific repo. |

## Secrets (all required)

| Secret | Purpose |
|---|---|
| `CLAUDE_CODE_ROLE_ARN` | IAM role ARN for OIDC; grants Bedrock invoke access |
| `GEMMA_CLAUDE_ASSISTANT_APP_ID` | GitHub App ID for bot identity |
| `GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY` | GitHub App private key (PEM) |

When using `secrets: inherit` inside Gemma Analytics org, these are already available at org level — no per-repo configuration needed.

---

## Review format

Claude posts:
- **Inline comments** on the diff — prefixed `[Blocking]`, `[Style]`, or `[Suggestion]`
- **Summary comment** on the PR timeline — one-sentence verdict, followed by grouped findings
- **Cost comment** — spend, turn count, duration, and model

All blocking issues found inline are also listed in the summary, so the author has one complete list without hunting through diff threads.

---

## Customising the review focus

The `additional_instructions` input is the primary customisation lever. Examples:

```yaml
additional_instructions: |
  - All dbt models must have a description in schema.yml
  - No hardcoded Snowflake schema names — use ref() and source()
  - Python functions over 50 lines must have a docstring
  - Never use SELECT * in production queries
```

To add a project-wide CLAUDE.md that the reviewer reads automatically, place one at the repo root. Claude reads it in Step 1 of every review and treats its rules as authoritative for the `[Style]` category.
