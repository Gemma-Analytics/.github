# Setting Up Claude Code Workflows in Your Organization

This guide walks you through setting up Gemma's Claude-powered GitHub workflows on your repositories. Once configured, Claude can automatically review every pull request, answer `@claude` mentions on issues and PRs, and run scheduled repository audits.

> **Working inside the Gemma-Analytics org?** You don't need this guide: the secrets are already configured at the org level and `secrets: inherit` works. Use the wrapper snippets in the [README quick start](../README.md#quick-start--adding-workflows-to-a-new-repo) instead. This guide is for **every other GitHub organization**, where secrets must be set up explicitly (see the [Secrets Reference](#secrets-reference) at the bottom).

## Contents

- [Who Does What](#who-does-what)
- [What You'll Need](#what-youll-need)
- [Admin Setup (Steps 1–2)](#admin-setup-steps-12)
  - [Step 1: Install the GitHub App](#step-1-install-the-github-app)
  - [Step 2: Add Secrets to Your GitHub Organization](#step-2-add-secrets-to-your-github-organization)
- [Developer Setup (Step 3)](#developer-setup-step-3)
  - [Step 3a: PR Reviews](#step-3a-pr-reviews-claude-reviewyml)
  - [Step 3b: `@claude` Mention Responder (Optional)](#step-3b-claude-mention-responder-optional)
  - [Step 3c: Scheduled Audits (Optional)](#step-3c-scheduled-audits-optional)
  - [Agent instructions](#agent-instructions)
- [Step 4: Test It](#step-4-test-it)
- [How It Works](#how-it-works)
- [Usage Limits](#usage-limits)
- [Version Pinning](#version-pinning)
- [Troubleshooting](#troubleshooting)
- [Secrets Reference](#secrets-reference)

## Who Does What

Before this guide reaches you, your Gemma contact has already prepared everything on their side (the AWS access role, the GitHub App, and your credentials). What remains is split into two parts with different access requirements:

| Part | Steps | Who needs to do it | Access required |
|------|-------|-------------------|-----------------|
| **Preparation** | *(already done)* | Your Gemma contact | — |
| **Admin setup** | Steps 1 & 2 | GitHub org admin / IT manager | GitHub Organization Owner |
| **Developer setup** | Steps 3 & 4 | Any developer with repo access | Repository write access |

> **Not a GitHub org admin?** Forward the section [Admin Setup (Steps 1–2)](#admin-setup-steps-12) below to the person who manages your GitHub organization, usually an IT manager or team lead. They can complete those steps independently, and you can proceed with Step 3 once they're done.

---

## What You'll Need

Your Gemma contact will provide these three items:

1. **Role ARN** — a string that looks like `arn:aws:iam::123456789012:role/github-actions-claude-code-yourcompany`
2. **App ID** — a number (e.g., `12345`)
3. **Private key file** — a `.pem` file (keep this safe, it's like a password)

*(For Gemma engineers: these come out of the client onboarding runbook, [admin-onboarding.md](admin-onboarding.md).)*

---

## Admin Setup (Steps 1–2)

> **This section requires GitHub Organization Owner access.** If you are not an org owner, ask your IT manager or GitHub admin to complete Steps 1 and 2. You can forward just this section to them; they don't need the rest of the guide.
>
> Once done, they should confirm which repositories have access so you can proceed to Step 3.

## Step 1: Install the GitHub App

1. Open the install link your Gemma contact shared with you
2. Select your GitHub organization
3. Choose which repositories should have access (you can select all or pick specific ones)
4. Click **Install**

That's it. The app is now installed on your org.

## Step 2: Add Secrets to Your GitHub Organization

> **Requires GitHub Organization Owner access** (the same person who completed Step 1).

Secrets are like passwords that GitHub Actions uses behind the scenes. You add them once at the organization level and every repository in your org can use them.

1. Go to your GitHub organization's page
2. Click **Settings** (top menu bar)
3. In the left sidebar, scroll down to **Secrets and variables** > **Actions**
4. Click **New organization secret**
5. Add these three secrets, one at a time:

| Name | Value |
|------|-------|
| `CLAUDE_CODE_ROLE_ARN` | Paste the Role ARN your Gemma contact gave you |
| `GEMMA_CLAUDE_ASSISTANT_APP_ID` | Paste the App ID number |
| `GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY` | Open the `.pem` file in a text editor, copy the entire contents, and paste it here |

For each secret, set **Repository access** to the repositories where you want Claude workflows enabled (or choose "All repositories").

> **Tip:** If setting org-level secrets isn't possible, you can add the same three secrets directly on each repository instead: go to the repository's **Settings** > **Secrets and variables** > **Actions**. This only requires repository **Admin** access (not org owner), which is useful if the org admin is unavailable.

See the [Secrets Reference](#secrets-reference) at the bottom of this guide for what each secret does and common pitfalls.

---

## Developer Setup (Step 3)

> **Requires repository write access.** Steps 1 and 2 must be complete before this step. Check with your admin that the GitHub App is installed and the three secrets have been added.

Each capability is a small "wrapper" workflow file you add to a repository. The wrapper calls the shared workflow hosted in `Gemma-Analytics/.github` and passes the three secrets along. Start with PR reviews (Step 3a); the mention responder and scheduled audits are optional add-ons that reuse the exact same secrets.

> **Important for every wrapper below:** the `secrets:` block at the bottom of each file is **mandatory**. Because the shared workflow lives in a different GitHub organization, `secrets: inherit` does not work — each wrapper must pass all three secrets explicitly by name. See the [Secrets Reference](#secrets-reference).

## Step 3a: PR Reviews (`claude-review.yml`)

Claude reviews every new pull request and posts inline comments plus a summary. You can also trigger a re-review any time by commenting `@claude review` on a PR.

### Option A — Let Claude Code do it (recommended)

If you have [Claude Code](https://claude.ai/code) available, it can inspect your repository, detect your tech stack, and write the workflow file for you, with the review instructions tailored to your codebase conventions.

**Before you start:** Make sure Steps 1 and 2 above are complete, and that you have Claude Code installed and are running it inside your repository folder.

**Paste this prompt into Claude Code:**

```
Set up Claude Code PR reviews for this repository.

Follow the agent instructions at:
https://github.com/Gemma-Analytics/.github/blob/main/docs/client-setup-guide.md#agent-instructions

Do not modify or assume the role ARN, App ID, or private key — those are
already configured as GitHub Actions secrets at the org or repo level.
```

Claude will inspect your repository, pick the right workflow template, customise the review instructions for your stack, and write `.github/workflows/claude-review.yml`. Then follow its instructions to commit and open a test PR.

---

### Option B — Paste the workflow file manually

In each repository where you want Claude reviews, create a file at this exact path:

```
.github/workflows/claude-review.yml
```

Paste this content into the file:

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
    secrets:
      CLAUDE_CODE_ROLE_ARN: ${{ secrets.CLAUDE_CODE_ROLE_ARN }}
      GEMMA_CLAUDE_ASSISTANT_APP_ID: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_ID }}
      GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY }}
```

Commit and push this file to the `main` branch.

> **Note:** The path `.github/.github/workflows/` is not a typo; it's how GitHub resolves reusable workflows from an organization's `.github` repository.

### Adding Review Instructions for Your Stack

You can tailor Claude's reviews to your tech stack by adding a `with` block. Here are examples for common repository types (drawn from live client setups):

**dbt transformations repo** (SQL models, tests, documentation):

```yaml
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
        - dbt SQL style conventions (leading commas, uppercase keywords, CTE naming)
        - Every model must have unique + not_null tests on its primary_key column
        - Proper use of {{ source() }} and {{ ref() }} — no hardcoded table references
        - Surrogate keys via dbt_utils.generate_surrogate_key()
        - Schema assignment required for every model in dbt_project.yml
        - No changes to archived or disabled models unless explicitly intended
    secrets:
      CLAUDE_CODE_ROLE_ARN: ${{ secrets.CLAUDE_CODE_ROLE_ARN }}
      GEMMA_CLAUDE_ASSISTANT_APP_ID: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_ID }}
      GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY }}
```

**Airflow / data pipeline repo** (DAGs, operators, dlt connectors):

```yaml
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
        - Proper Airflow DAG structure (default_args, catchup=False, tags)
        - No hardcoded connection strings or credentials — use Airflow Variables/Connections
        - Task dependencies should be explicit and acyclic
        - Correct use of operators (no BashOperator for Python tasks)
        - Idempotent tasks — re-running a task should not cause duplicate data
        - All secrets must come from environment variables or secret managers; never hardcoded
        - Python must be run via uv (uv run, uv sync); never bare python or pip
    secrets:
      CLAUDE_CODE_ROLE_ARN: ${{ secrets.CLAUDE_CODE_ROLE_ARN }}
      GEMMA_CLAUDE_ASSISTANT_APP_ID: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_ID }}
      GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY }}
```

**Infrastructure repo** (OpenTofu/Terraform, Ansible, Snowflake access control):

```yaml
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
        - Proper use of OpenTofu conventions (not Terraform)
        - No hardcoded credentials, tokens, or account identifiers
        - Proper Permifrost YAML structure and Snowflake access control patterns
        - Environment isolation (sandbox vs prod configurations)
        - Correct Ansible playbook structure and variable handling
    secrets:
      CLAUDE_CODE_ROLE_ARN: ${{ secrets.CLAUDE_CODE_ROLE_ARN }}
      GEMMA_CLAUDE_ASSISTANT_APP_ID: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_ID }}
      GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY }}
```

You can mix and match these instructions or write your own. The `additional_instructions` field accepts any free-text guidance that helps Claude understand your codebase conventions. For all other inputs (model, region, max turns), see [claude-review.md](claude-review.md).

## Step 3b: `@claude` Mention Responder (Optional)

Beyond reviews, Claude can respond to `@claude` mentions anywhere in your repos: it can answer questions on issues, help in PR discussions, and reply to inline review comments.

Create `.github/workflows/claude.yml`:

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
    secrets:
      CLAUDE_CODE_ROLE_ARN: ${{ secrets.CLAUDE_CODE_ROLE_ARN }}
      GEMMA_CLAUDE_ASSISTANT_APP_ID: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_ID }}
      GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY }}
```

> **Why the `!startsWith(..., '@claude review')` conditions?** They route `@claude review` comments to the review workflow (Step 3a) instead of the generic responder. If you use both workflows, keep the two `if:` conditions in sync (see [claude.md](claude.md) for the full routing rules). Note that the trigger is case-sensitive: `@claude review` triggers a review, `@Claude review` does not.

## Step 3c: Scheduled Audits (Optional)

Once the secrets are in place (Steps 1–2), you can also enable automated monthly audits; no additional secrets or access control changes are needed. Each audit run checks out your repository read-only, writes a dated Markdown report under `docs/audit-reports/`, and opens a PR so your team can review and act on the findings. Running the same audit twice on one date updates the existing branch and PR rather than creating a duplicate.

Two audit types are available today (see [claude-audit.md](claude-audit.md) for details and how to add more):

- **Security audit** — scans Docker Compose files, Dockerfiles, CI/CD workflows, infrastructure-as-code, Airflow config, and pre-commit hooks for security issues.
- **dbt audit** — validates a dbt repository against modelling, testing, and documentation conventions.

### Security audit

Create `.github/workflows/scheduled-audit.yml`:

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
    # Defaults run the deployment-security audit — no `with:` block needed.
    uses: Gemma-Analytics/.github/.github/workflows/claude-audit.yml@main
    secrets:
      CLAUDE_CODE_ROLE_ARN: ${{ secrets.CLAUDE_CODE_ROLE_ARN }}
      GEMMA_CLAUDE_ASSISTANT_APP_ID: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_ID }}
      GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY }}
```

### dbt audit

For dbt repositories, create `.github/workflows/dbt-audit.yml`:

```yaml
name: dbt Repo Audit

on:
  schedule:
    - cron: "0 7 1 * *"   # 07:00 UTC on the 1st (offset from the security audit)
  workflow_dispatch:

permissions:
  id-token: write
  contents: write
  pull-requests: write

jobs:
  audit:
    uses: Gemma-Analytics/.github/.github/workflows/claude-audit.yml@main
    with:
      plugin: gemma-dbt
      orchestrator_skill: validate-repo
      report_slug: dbt
      pr_title_prefix: "📊 dbt audit"
      # validate-repo fans out parallel sub-agents → allow the Task tool.
      allowed_tools: "Task,Bash(find *),Bash(grep *),Bash(cat *),Bash(ls *),Bash(dbt parse),Read,Glob,Grep,Write"
      extra_instructions: >-
        Automated, unattended run. Do not ask any questions. dbt is not installed
        in this runner: let `dbt parse` fail and proceed in filename-only mode,
        noting the degradation in the report.
    secrets:
      CLAUDE_CODE_ROLE_ARN: ${{ secrets.CLAUDE_CODE_ROLE_ARN }}
      GEMMA_CLAUDE_ASSISTANT_APP_ID: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_ID }}
      GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY }}
```

Commit either file on a branch and merge to `main`. The monthly cron fires automatically from then on; to run an audit immediately, go to **Actions** → select the audit workflow → **Run workflow**.

> **Tips:**
> - If you enable both audits in one repo, offset the cron schedules (as shown above) so they don't compete for the usage budget in the same 3-hour window.
> - For unattended scheduled runs, consider pinning to a commit SHA instead of `@main` for reproducibility (see [Version Pinning](#version-pinning)).

---

### Agent instructions

> *This section is written for Claude Code. If you're a human, you can skip it; the prompt in Option A above handles everything.*

You are setting up Claude Code PR reviews for the current repository. Follow these steps precisely.

**1. Detect the repository type**

Check for these files/directories at the repo root:

- `dbt_project.yml` → **dbt** repo
- `dags/` directory → **Airflow** repo
- A `pyproject.toml` that imports `dlt` → **dlt pipeline** repo
- `*.tf` files or `ansible.cfg` / `inventory*` → **infrastructure** repo
- None of the above → use the **generic** template (bare minimum workflow, no `additional_instructions`)

A repo may match more than one type (e.g., dbt + Airflow). If so, combine `additional_instructions` from both templates.

**2. Read project conventions**

If a `CLAUDE.md` file exists at the repo root, read it and extract:

- SQL style rules (comma placement, keyword casing, line length)
- Required tests (e.g., "every model needs `not_null` on its PK")
- Naming conventions (surrogate key suffixes, schema assignment rules)
- Forbidden patterns ("never use BashOperator", "always use `uv run`")
- Any other concrete, diff-verifiable rules

These become the bullet points in `additional_instructions`. Keep them specific: "every model must have a `not_null` test on its primary_key column" is good; "write high-quality models" is not.

**3. Write the workflow file**

Create `.github/workflows/claude-review.yml` with this structure:

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
        <insert bullet points derived from repo type + CLAUDE.md conventions>
    secrets:
      CLAUDE_CODE_ROLE_ARN: ${{ secrets.CLAUDE_CODE_ROLE_ARN }}
      GEMMA_CLAUDE_ASSISTANT_APP_ID: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_ID }}
      GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY }}
```

**Critical rules:**

- **Never omit or change the `secrets:` block.** The reusable workflow lives in a different org (`Gemma-Analytics`) and `secrets: inherit` does not work cross-org. If any of the three secrets is missing, the workflow fails with "Secret X is required, but not provided". Pass all three explicitly.
- **Never change the `permissions:` block.** All four permissions are required: `id-token: write` for AWS OIDC, `pull-requests: write` to post review comments, `issues: write` for the `@claude review` comment trigger, `contents: read` to read the diff.
- **Never change the `on:` triggers or the `if:` condition.** Both PR events and comment triggers are required for the full feature set (automatic review on open + on-demand re-review via `@claude review`).
- **Do not assume or hardcode the role ARN, App ID, or private key.** Those are already set as GitHub Actions secrets by the repository owner.
- If the repo has no `CLAUDE.md` and no clear tech stack, omit the `with:` block entirely (bare minimum workflow).
- If the user also asks for the `@claude` mention responder or a scheduled audit, use the exact templates from [Step 3b](#step-3b-claude-mention-responder-optional) and [Step 3c](#step-3c-scheduled-audits-optional) of this guide. The same critical rules apply (explicit `secrets:` block, unmodified `permissions:` and `if:` conditions). The mention responder's `if:` must keep the `!startsWith(..., '@claude review')` negations so review requests route to the review workflow.

**4. Tell the user what to do next**

After writing the file, tell the user:

1. Commit the file on a feature branch and push it
2. Merge to `main` (or whichever the default branch is); the workflow must be on the default branch to trigger on PRs
3. Open a small test PR (any trivial change is fine)
4. Check the **Actions** tab: a `Claude Code Review` run should appear within ~1 minute
5. Once complete, Claude's review comment will appear on the PR

If the Actions run fails, the most common causes are:
- The three secrets are not set at the org or repo level (check **Settings → Secrets and variables → Actions**)
- The GitHub App is not installed on this repository (check **Settings → GitHub Apps** or the app's own settings page)

---

## Step 4: Test It

1. Create a new branch in your repository
2. Make a small code change
3. Open a pull request

Within a few minutes, Claude will post a review comment on your PR. If it doesn't appear after 5 minutes, check the **Actions** tab in your repository for any error messages.

You can also trigger a re-review at any time by commenting `@claude review` on any open PR.

## How It Works

When you open or update a pull request, GitHub Actions runs the review workflow:

1. GitHub sends a secure, temporary token to AWS (no passwords are stored)
2. AWS verifies the token and provides short-lived access to the Claude AI model
3. Claude reviews your code changes and posts comments on the PR
4. The temporary access expires after one hour

Your code is processed by Claude for the review and is not stored afterward. The `@claude` mention responder and scheduled audits work the same way, with the same authentication and the same short-lived access.

## Usage Limits

Your organization has a usage budget to keep costs predictable. If the budget is reached, reviews will pause temporarily and resume automatically when the usage window resets (every 3 hours, daily, or weekly depending on which limit was hit).

You'll receive an email notification at the address your Gemma contact has on file if a limit is reached.

## Version Pinning

The snippets above reference the shared workflows with `@main`, which means you automatically get improvements as they're released. If you prefer a stable pin (recommended for unattended scheduled audits), reference a specific commit SHA instead:

```yaml
uses: Gemma-Analytics/.github/.github/workflows/claude-audit.yml@<commit-sha>
```

You can find the current SHA on the [Gemma-Analytics/.github commits page](https://github.com/Gemma-Analytics/.github/commits/main). Update the pin deliberately when you want to pick up new behaviour.

## Troubleshooting

**Claude doesn't post a review**
- Check the **Actions** tab in your repository for failed workflow runs
- Make sure the workflow file is on the `main` (or default) branch
- Verify the GitHub App is installed on the repository

**Workflow fails with "Not authorized" error**
- Double-check the three secrets are set correctly (no extra spaces or newlines)
- Make sure the `.pem` private key was copied in full (including the `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----` lines)

**Workflow fails with "Secret X is required, but not provided"**
- The workflow must pass secrets explicitly, because `secrets: inherit` does not work across organizations
- Verify the workflow file contains the explicit `secrets:` block shown in the templates above
- Check that org secrets have **Repository access** set to "All repositories" or the specific repo

**Workflow fails with "could not find reusable workflow" error**
- This usually means a network issue. Re-run the failed workflow from the Actions tab.

**`@claude` mention gets a review instead of an answer (or vice versa)**
- Routing is based on the comment prefix: comments starting with `@claude review` go to the review workflow; all other `@claude` mentions go to the responder
- The trigger is case-sensitive: `@Claude review` will not start a review
- Check that the `if:` conditions in both wrapper files match the templates exactly

**Need help?**
Contact your Gemma point of contact, or email support at the address they provided during onboarding.

---

## Secrets Reference

### Inside Gemma-Analytics vs. any other organization

This is the single most important difference when setting up these workflows outside the Gemma-Analytics org:

| | Repos in `Gemma-Analytics` | Repos in **any other org** |
|---|---|---|
| Secret values | Already configured at the org level, nothing to set up | Must be **created in your org** (Step 2 above), at the org level or per repo |
| Wrapper `secrets:` | `secrets: inherit` works | Must **pass all three explicitly by name**; `secrets: inherit` silently passes nothing across org boundaries |

Every wrapper in this guide therefore ends with this exact block:

```yaml
    secrets:
      CLAUDE_CODE_ROLE_ARN: ${{ secrets.CLAUDE_CODE_ROLE_ARN }}
      GEMMA_CLAUDE_ASSISTANT_APP_ID: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_ID }}
      GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY }}
```

If you omit it (or use `secrets: inherit`), the run fails with *"Secret X is required, but not provided"*.

### The three secrets

| Secret | What it is | Where it comes from |
|---|---|---|
| `CLAUDE_CODE_ROLE_ARN` | AWS IAM role ARN; lets GitHub Actions authenticate to AWS Bedrock via OIDC (no long-lived AWS keys) | Provided by your Gemma contact, provisioned per client org |
| `GEMMA_CLAUDE_ASSISTANT_APP_ID` | GitHub App ID; comments and PRs appear under the **Gemma Claude Assistant** bot identity | Provided by your Gemma contact (same for all clients) |
| `GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY` | GitHub App private key (`.pem` file contents) | Provided by your Gemma contact, generated per client for independent revocation |

All three workflows (reviews, mentions, audits) share the same three secrets, so enabling an additional workflow never requires new secrets or AWS changes.

### Common pitfalls

- **Paste the `.pem` file whole.** The private key secret must contain the complete multi-line file, including the `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----` lines, with all line breaks preserved. A key collapsed onto a single line fails at runtime with `ERR_OSSL_UNSUPPORTED`.
- **Grant repository access.** Org-level secrets have a **Repository access** setting. If a repo isn't covered ("All repositories" or explicitly selected), the workflow behaves as if the secret doesn't exist.
- **Org secrets need a GitHub plan that supports them** for private repos (GitHub Free orgs only expose org secrets to public repos). If org-level secrets aren't available, fall back to per-repo secrets: same names, set in each repository's **Settings** > **Secrets and variables** > **Actions**.
- **Never commit these values.** The role ARN, App ID, and key belong only in GitHub Actions secrets, never in the workflow file, the repo, or chat logs.
