# `claude.yml` — General-purpose `@claude` Mention Responder

Handles any `@claude` mention in issues, pull request comments, and PR review threads — except `@claude review`, which routes to `claude-review.yml`. Claude can answer questions, help draft text, suggest fixes, or perform any other task described in the comment.

---

## Architecture

```
Client repo                       Gemma-Analytics/.github              AWS / GitHub
─────────────                     ────────────────────────             ─────────────
.github/workflows/
  claude.yml
    │
    │  on: issue_comment           claude.yml (reusable)
    │  on: pull_request_review     ┌─────────────────────────────────────────────────┐
    │     _comment                 │ 1. Checkout client repo                         │
    │  on: issues (opened,         │                                                 │
    │       assigned)              │ 2. Create GitHub App token ────────────────────▶│ Gemma Claude
    └──── workflow_call ──────────▶│    (Gemma Claude Assistant)                     │ Assistant App
                                   │                                                 │
                                   │ 3. OIDC → assume IAM role ─────────────────────▶│ AWS Bedrock
                                   │    (CLAUDE_CODE_ROLE_ARN)                       │ (eu-central-1)
                                   │                                                 │
                                   │ 4. React 👀 to trigger comment                  │
                                   │    (if comment event)                           │
                                   │                                                 │
                                   │ 5. claude-code-action@v1 ──────────────────────▶│ Claude
                                   │    ├─ reads trigger context natively            │ (Bedrock)
                                   │    ├─ handles @claude mention automatically     │
                                   │    └─ responds in the same thread               │
                                   │                                                 │
                                   │ 6. Post session cost comment                    │
                                   └─────────────────────────────────────────────────┘
```

### Trigger routing

```
Comment: "@claude review" ──────────────────────────────────────────▶ routes to claude-review.yml
Comment: "@claude review focus on security" ────────────────────────▶ routes to claude-review.yml

Comment: "@claude what does this function do?" ─────────────────────▶ claude.yml handles it
Comment: "@claude please help me fix this" ─────────────────────────▶ claude.yml handles it
Comment: "@claude please review this" ──────────────────────────────▶ claude.yml handles it
                                                                       (doesn't START with "@claude review")
Issue body contains "@claude" ──────────────────────────────────────▶ claude.yml handles it
PR review thread contains "@claude" ────────────────────────────────▶ claude.yml handles it
```

> `@claude review` routing is **case-sensitive**. `@Claude review` (capital C) routes to the generic handler.

---

## Client wrapper

Create `.github/workflows/claude.yml` in the client repo:

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

The `!startsWith(..., '@claude review')` guard ensures `@claude review` always routes to the review workflow, not here. Keep both wrapper files consistent on this guard if you ever modify the routing logic.

---

## Inputs

| Input | Default | Description |
|---|---|---|
| `model` | `eu.anthropic.claude-sonnet-4-6` | Bedrock model ID. |
| `aws_region` | `eu-central-1` | AWS region for Bedrock OIDC. |
| `prompt` | _(empty)_ | When empty, the action reads the trigger event (comment/issue body) and handles `@claude` mentions natively. Pass an explicit prompt only for special routing scenarios — rarely needed in practice. |
| `max_turns` | _(action default)_ | When blank, the action uses its built-in default. Set a number to cap cost on repos with expensive interactions. |

## Secrets (all required)

| Secret | Purpose |
|---|---|
| `CLAUDE_CODE_ROLE_ARN` | IAM role ARN for OIDC; grants Bedrock invoke access |
| `GEMMA_CLAUDE_ASSISTANT_APP_ID` | GitHub App ID for bot identity |
| `GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY` | GitHub App private key (PEM) |

When using `secrets: inherit` inside Gemma Analytics org, these are available at org level.

---

## What Claude can do via `@claude`

By default the action has broad tool access (full tool set). Typical tasks that work well:

- Answer questions about the codebase (`@claude explain this function`)
- Help write PR descriptions or issue bodies
- Suggest a fix for a specific error shown in a comment
- Draft a migration or refactor plan based on an issue description
- Clarify what a failing test is checking

Claude responds directly in the GitHub comment thread where it was mentioned.

---

## Limiting tool access (optional)

If you want to restrict what Claude can do in `@claude` mentions, pass `claude_args` via the `prompt` input approach or open a PR to extend the reusable workflow with a `claude_args` input. Currently not exposed as an input because most repos benefit from the full default tool set.
