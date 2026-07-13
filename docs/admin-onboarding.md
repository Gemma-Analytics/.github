# Client Onboarding — Internal Guide

**Audience: Gemma engineers.** How to onboard a new external client for the Claude Code workflows on their GitHub org.

Once onboarding is complete, the client can use all three reusable workflows from this repository:

- [`claude-review.yml`](claude-review.md) — reviews every new PR automatically; comment `@claude review` to re-run it on demand
- [`claude.yml`](claude.md) — responds to `@claude` mentions in issues, PR comments, and inline review comments
- [`claude-audit.yml`](claude-audit.md) — runs a monthly repository audit (deployment-security or dbt) and opens a PR with a dated report

**Time:** ~30 minutes per client (after initial setup).

## Contents

- [How Onboarding Works](#how-onboarding-works)
- [Phase 1 — Gemma Engineer Prepares Everything](#phase-1--gemma-engineer-prepares-everything)
  - [Prerequisites](#prerequisites)
  - [Step 1: Provision the AWS IAM role (gemma-infrastructure)](#step-1-provision-the-aws-iam-role-gemma-infrastructure)
  - [Step 2: Set Bedrock quotas (gemma-infrastructure)](#step-2-set-bedrock-quotas-gemma-infrastructure)
  - [Step 3: Generate the GitHub App private key](#step-3-generate-the-github-app-private-key)
  - [Step 4: Store the credentials in the client's 1Password vault](#step-4-store-the-credentials-in-the-clients-1password-vault)
  - [Step 5: Hand off to the client](#step-5-hand-off-to-the-client)
- [Phase 2 — Client Org Admin Sets Up GitHub](#phase-2--client-org-admin-sets-up-github)
- [Phase 3 — Client Developers Enable the Workflows](#phase-3--client-developers-enable-the-workflows)
- [Troubleshooting](#troubleshooting)

## How Onboarding Works

Onboarding is split into three phases, each with a clear owner. Everything in Phase 1 is prepared by you (the Gemma engineer) **before** the client is involved; Phases 2 and 3 happen on the client's side, following the guide you hand them.

| Phase | Who | What | Documented in |
|-------|-----|------|---------------|
| **1. Prepare** | Gemma engineer | AWS IAM role + Bedrock quota (PR in gemma-infrastructure), GitHub App private key, credentials in the client's 1Password vault, handoff package | This guide |
| **2. GitHub setup** | Client's GitHub org admin | Install the GitHub App, add the three secrets to their org | [Client Setup Guide — Admin Setup](client-setup-guide.md#admin-setup-steps-12) |
| **3. Enable workflows** | Client's developers | Add the wrapper workflow files to each repo | [Client Setup Guide — Developer Setup](client-setup-guide.md#developer-setup-step-3) |

The work is also split across two repositories: **AWS/Bedrock provisioning (Steps 1–2) is Terraform and lives in [gemma-infrastructure](https://github.com/Gemma-Analytics/gemma-infrastructure)**, where it is documented; the GitHub-only steps (3–5) are documented here.

---

## Phase 1 — Gemma Engineer Prepares Everything

### Prerequisites

- The shared **Gemma Claude Assistant** GitHub App must be public. If not yet done:
  1. Go to `https://github.com/organizations/Gemma-Analytics/settings/apps`
  2. Click **Gemma Claude Assistant** > **Advanced** (sidebar)
  3. In the "Danger zone" section, click **Make public**

### Step 1: Provision the AWS IAM role (gemma-infrastructure)

Open a PR in the gemma-infrastructure repo adding the client to the `claude_code_clients` tfvars (this creates the `github-actions-claude-code-<slug>` IAM role with an OIDC trust policy for the client's GitHub org), apply sandbox → prod, and note the resulting **role ARN**.

➡️ **Follow: [Client AWS Provisioning — Step 1: Add the client IAM role](https://github.com/Gemma-Analytics/gemma-infrastructure/blob/main/aws/account/cicd/docs/client-aws-provisioning.md#step-1-add-the-client-iam-role)**

> ⚠️ The client's `github_org` value must use the exact canonical casing (`gh api orgs/<org-name> --jq '.login'`), because OIDC `sub` claims are case-sensitive. Details in the linked doc.

### Step 2: Set Bedrock quotas (gemma-infrastructure)

In the same (or a follow-up) gemma-infrastructure PR, add the new role to the Bedrock `role_quotas` tfvars. This caps the client's spend per 3-hour/day/week window and sets the alert email. Pick a tier from the quota reference table; factor in scheduled audits if the client will enable them.

➡️ **Follow: [Client AWS Provisioning — Step 2: Set Bedrock quotas](https://github.com/Gemma-Analytics/gemma-infrastructure/blob/main/aws/account/cicd/docs/client-aws-provisioning.md#step-2-set-bedrock-quotas)**

### Step 3: Generate the GitHub App private key

Each client gets their own private key for the shared GitHub App, so one client's access can be revoked without affecting the others.

1. Go to `https://github.com/organizations/Gemma-Analytics/settings/apps`
2. Click **Gemma Claude Assistant**
3. Under **Private keys**, click **Generate a private key**
4. Download the `.pem` file
5. Note the fingerprint and label it with the client name for tracking

**To revoke a client later:** Delete their private key from the app settings. Their workflows will fail to generate tokens; other clients are unaffected.

### Step 4: Store the credentials in the client's 1Password vault

Create an item in the **client's vault** named `<Client> - Gemma Claude App credentials (PR Review bot)` containing everything the client will need:

- The **`.pem` private key as a file attachment** (from Step 3). Do not paste the key contents as a text field, because file attachments preserve the formatting more reliably.
- The **role ARN** (from Step 1) as a text field
- The **App ID** (visible on the app's settings page; shared across all clients) as a text field

**When sharing the key with the client:** instruct them to paste the raw file contents into the GitHub Actions secret exactly as-is, preserving all line breaks. The key must appear in the secret as multiple lines like this:

```
-----BEGIN RSA PRIVATE KEY-----
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
(... many lines ...)
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
-----END RSA PRIVATE KEY-----
```

A single-line or reformatted key will fail with `ERR_OSSL_UNSUPPORTED` at runtime.

### Step 5: Hand off to the client

Share the [Client Setup Guide](client-setup-guide.md) with your client contact, along with the four items below (via the 1Password item from Step 4, or your agreed secure channel):

- Their **role ARN** (from Step 1)
- The **App ID** (shared across all clients; visible on the app's settings page)
- Their **private key** `.pem` file (from Step 3)
- The **install URL**: `https://github.com/apps/gemma-claude-assistant/installations/new`

Point them at the [Who Does What](client-setup-guide.md#who-does-what) section so they route the admin steps to their GitHub org owner.

---

## Phase 2 — Client Org Admin Sets Up GitHub

Done by the client's GitHub Organization Owner (or IT manager), using the guide and items you handed over:

1. **Install the GitHub App** on their org → [Client Setup Guide — Step 1](client-setup-guide.md#step-1-install-the-github-app)
2. **Add the three secrets** (`CLAUDE_CODE_ROLE_ARN`, `GEMMA_CLAUDE_ASSISTANT_APP_ID`, `GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY`) at their org level → [Client Setup Guide — Step 2](client-setup-guide.md#step-2-add-secrets-to-your-github-organization)

Because the client is a different GitHub org, `secrets: inherit` does not work: the secrets must exist in **their** org, and every wrapper must pass them explicitly. This is spelled out in the guide's [Secrets Reference](client-setup-guide.md#secrets-reference).

## Phase 3 — Client Developers Enable the Workflows

Done by any client developer with repository write access, per repo:

1. Add the wrapper workflow file(s) → [Client Setup Guide — Step 3](client-setup-guide.md#developer-setup-step-3) (PR reviews, optionally `@claude` mentions and scheduled audits)
2. Open a test PR → [Client Setup Guide — Step 4](client-setup-guide.md#step-4-test-it)

Once the first review lands, onboarding is complete.

---

## Troubleshooting

**Workflow fails with "Secret X is required, but not provided while calling"**
- Client workflows must pass secrets explicitly, because `secrets: inherit` does not work cross-org
- Verify the workflow uses explicit `secrets:` mapping (see the [client setup guide examples](client-setup-guide.md#secrets-reference))
- Check that org secrets have **Repository access** set to "All repositories" or the specific repo

**Workflow fails with "Not authorized" when generating the App token**
- The `.pem` key was likely pasted reformatted or single-line; [Step 4](#step-4-store-the-credentials-in-the-clients-1password-vault) shows the required format
- Verify the GitHub App is installed on the repository

**AWS-side failures** (`sts:AssumeRoleWithWebIdentity`, quota alerts, dashboard, CI/CD plan permissions) → see [Client AWS Provisioning — Troubleshooting (AWS)](https://github.com/Gemma-Analytics/gemma-infrastructure/blob/main/aws/account/cicd/docs/client-aws-provisioning.md#troubleshooting-aws).

**Client-visible issues** (no review posted, routing, workflow not found) → see the [client setup guide's troubleshooting section](client-setup-guide.md#troubleshooting).
