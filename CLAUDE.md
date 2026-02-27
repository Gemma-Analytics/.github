# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

Organization-level `.github` repository for Gemma Analytics. Contains shared GitHub Actions workflows used across all Gemma repos. This is **not** an application — it is infrastructure configuration.

## Reusable Workflows

### `claude.yml` — Interactive Claude sessions

Called by repo workflows when someone mentions `@claude` in a PR or issue comment. Handles:
- GitHub App token generation (`gemma-claude-assistant`)
- AWS OIDC for Bedrock access
- Comment acknowledgment (eyes reaction)
- Session cost reporting

### `claude-review.yml` — Automated PR reviews

Called by repo workflows on `pull_request` events (`opened`, `synchronize`). Bakes in a review prompt that:
- Reads and follows each repo's CLAUDE.md
- Reviews for correctness, security, performance, style, and test coverage
- Accepts `additional_instructions` for repo-specific checks
- Allows `git`, `grep`, and `find` via `--allowedTools`
- Posts "Claude review cost" to distinguish from interactive sessions

## How Repos Use These Workflows

Each repo has thin caller workflows in `.github/workflows/`:

```yaml
# .github/workflows/claude.yml (interactive)
jobs:
  claude:
    uses: Gemma-Analytics/.github/.github/workflows/claude.yml@main
    secrets: inherit

# .github/workflows/claude-review.yml (automated review)
jobs:
  review:
    uses: Gemma-Analytics/.github/.github/workflows/claude-review.yml@main
    with:
      additional_instructions: |
        - Repo-specific review checks...
    secrets: inherit
```

## Required Secrets (set at org level)

| Secret | Purpose |
|--------|---------|
| `CLAUDE_CODE_ROLE_ARN` | IAM role ARN for AWS Bedrock access via OIDC |
| `GEMMA_CLAUDE_ASSISTANT_APP_ID` | GitHub App ID for `gemma-claude-assistant` |
| `GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY` | GitHub App private key |

## Conventions

- Changes must go through a pull request — never commit directly to `main`
- The `gemma-claude-assistant` GitHub App must be installed on any repo that uses these workflows
- Default model is `eu.anthropic.claude-sonnet-4-6` (EU region Bedrock)
- Workflow files use standard GitHub Actions YAML conventions
- Keep workflows minimal — avoid adding repo-specific logic here
