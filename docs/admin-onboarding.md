# Client Onboarding — Internal Guide

How to onboard a new external client for Claude Code PR reviews on their GitHub org.

**Time:** ~30 minutes per client (after initial setup).

> **Note:** The infrastructure steps below (Steps 1–2) are executed in the [gemma-infrastructure](https://github.com/Gemma-Analytics/gemma-infrastructure) repository — all `tofu` commands and tfvars paths refer to that repo, not this one. This doc lives here so it sits next to the [client setup guide](client-setup-guide.md) you hand to the client at the end.

## Prerequisites

- The shared **Gemma Claude Assistant** GitHub App must be public. If not yet done:
  1. Go to `https://github.com/organizations/Gemma-Analytics/settings/apps`
  2. Click **Gemma Claude Assistant** > **Advanced** (sidebar)
  3. In the "Danger zone" section, click **Make public**

## Step 1: Add Client to `aws/account/cicd/` Module

Add an entry to `claude_code_clients` in **both** [`aws/account/cicd/environments/prod.tfvars`](https://github.com/Gemma-Analytics/gemma-infrastructure/blob/main/aws/account/cicd/environments/prod.tfvars) and [`aws/account/cicd/environments/sandbox.tfvars`](https://github.com/Gemma-Analytics/gemma-infrastructure/blob/main/aws/account/cicd/environments/sandbox.tfvars):

```hcl
claude_code_clients = {
  # ... existing clients ...
  <client-slug> = {
    github_org      = "<ClientGitHubOrg>"
    allow_all_repos = true
    # Or restrict to specific repos:
    # repos = ["repo-a", "repo-b"]
  }
}
```

The `<client-slug>` becomes part of the IAM role name (`github-actions-claude-code-<slug>`) and must be a valid IAM role name suffix (lowercase, hyphens OK, no spaces).

**Important: `github_org` must use the exact canonical casing.** GitHub OIDC `sub` claims are case-sensitive, but GitHub URLs are not — so `github.com/saxonia-catering` works in a browser but `saxonia-catering` will fail in the trust policy if the canonical name is `Saxonia-Catering`. Always verify with:

```bash
gh api orgs/<org-name> --jq '.login'
```

### Apply

In the gemma-infrastructure repo, test in sandbox first:

```bash
cd aws/account/cicd
tofu init -reconfigure -backend-config=backends/sandbox.hcl
tofu plan -var-file=environments/sandbox.tfvars
tofu apply -var-file=environments/sandbox.tfvars
```

Then production (via PR + `/apply` comment, or locally):

```bash
tofu init -reconfigure -backend-config=backends/prod.hcl
AWS_PROFILE=infra-prod tofu plan -var-file=environments/prod.tfvars
AWS_PROFILE=infra-prod tofu apply -var-file=environments/prod.tfvars
```

### Verify

```bash
# Get the client's role ARN
tofu output -json claude_code_client_role_arns

# Check trust policy is scoped correctly
aws iam get-role \
  --role-name github-actions-claude-code-<slug> \
  --query 'Role.AssumeRolePolicyDocument' \
  --profile infra-sandbox
```

## Step 2: Add Client Quotas to `aws/account/bedrock/` Module

Add the client's role to `role_quotas` in **both** [`aws/account/bedrock/environments/prod.tfvars`](https://github.com/Gemma-Analytics/gemma-infrastructure/blob/main/aws/account/bedrock/environments/prod.tfvars) and [`aws/account/bedrock/environments/sandbox.tfvars`](https://github.com/Gemma-Analytics/gemma-infrastructure/blob/main/aws/account/bedrock/environments/sandbox.tfvars):

```hcl
role_quotas = {
  # ... existing roles ...
  github-actions-claude-code-<slug> = {
    per_3h   = 3.0    # USD per 3-hour window
    per_day  = 6.0    # USD per day
    per_week = 10.0   # USD per week
    email    = "contact@client-domain.com"  # quota alert recipient
  }
}
```

Use the sandbox values at ~10% of prod for testing.

The quota enforcer Lambda and CloudWatch dashboard automatically pick up new `role_quotas` entries — no code changes needed.

### Apply

In the gemma-infrastructure repo:

```bash
cd aws/account/bedrock
tofu init -reconfigure -backend-config=backends/sandbox.hcl
tofu plan -var-file=environments/sandbox.tfvars
tofu apply -var-file=environments/sandbox.tfvars
```

Then production (via PR + `/apply`).

## Step 3: Generate a Client Private Key

Each client gets their own private key for the shared GitHub App. This allows independent revocation.

1. Go to `https://github.com/organizations/Gemma-Analytics/settings/apps`
2. Click **Gemma Claude Assistant**
3. Under **Private keys**, click **Generate a private key**
4. Download the `.pem` file
5. Note the fingerprint — label it with the client name for tracking
6. **Store the `.pem` file as a file attachment** in the client's `<Client> - Gemma Claude App credentials (PR Review bot)` 1Password item (in the client's vault), labelled with the client name. Do not paste the key contents as a text field — file attachments preserve formatting reliably.

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

**To revoke a client later:** Delete their private key from the app settings. Their workflows will fail to generate tokens; other clients are unaffected.

## Step 3b: Bedrock quota — account for scheduled audits

If the client will also run scheduled audits (see [client-setup-guide.md — Scheduled Audits](client-setup-guide.md#step-3c-scheduled-audits-optional)), factor in the monthly audit cost when choosing a quota tier. A full `deployment-security` audit on a typical Airflow/Docker repo costs roughly $0.10–$0.30 per run with Sonnet — negligible against weekly quotas, but worth noting at the Low tier.

No separate IAM role, OIDC trust policy, or bedrock quota entry is needed for audits — the audit workflow uses the same `github-actions-claude-code-<slug>` role already provisioned. The audit's OIDC `sub` claim is `repo:<org>/<repo>:ref:refs/heads/main`, which is covered by the existing `allow_all_repos = true` trust policy.

## Step 4: Client Setup

Share the [Client Setup Guide](client-setup-guide.md) with the client, along with:

- Their **role ARN** (from step 1)
- The **App ID** (shared across all clients — visible on the app's settings page)
- Their **private key** `.pem` file (from step 3)
- The **install URL**: `https://github.com/apps/gemma-claude-assistant/installations/new`

## Quota Reference

| Tier | per_3h | per_day | per_week | Use case |
|------|--------|---------|----------|----------|
| Low | $3 | $6 | $10 | Small teams, light review usage |
| Standard | $8 | $15 | $25 | Same as Gemma internal |
| High | $12 | $25 | $50 | Large teams, heavy usage |

Derive from the weekly budget: `per_day ≈ 60%`, `per_3h ≈ 30%`.

## Troubleshooting

**Workflow fails with "Secret X is required, but not provided while calling"**
- Client workflows must pass secrets explicitly — `secrets: inherit` does not work cross-org
- Verify the workflow uses explicit `secrets:` mapping (see the [client setup guide examples](client-setup-guide.md#secrets-reference))
- Check that org secrets have **Repository access** set to "All repositories" or the specific repo

**Client workflow fails with "Not authorized to perform sts:AssumeRoleWithWebIdentity"**
- Check the trust policy allows the client's GitHub org: `aws iam get-role --role-name github-actions-claude-code-<slug> --query 'Role.AssumeRolePolicyDocument'`
- Verify the org name casing matches exactly: `gh api orgs/<org-name> --jq '.login'`
- GitHub OIDC `sub` claims use canonical casing (e.g., `Saxonia-Catering` not `saxonia-catering`) — the trust policy must match
- If using `repos` instead of `allow_all_repos`, verify the repo name is correct

**Quota alerts not sending to client email**
- Verify the email address is in `role_quotas` for this role
- Check SES: the email domain may need verification if sending from a non-sandbox SES region
- Check Lambda logs: `aws logs tail /aws/lambda/bedrock-quota-enforcer --follow`

**Client shows up as "unknown" in dashboard**
- The role must be in `role_quotas` in bedrock tfvars for the dashboard to track it
- It can take up to 15 minutes for the first metrics to appear after a deploy

**CI/CD plan fails with "not authorized to perform iam:GetOpenIDConnectProvider"**
- The `github-actions-bedrock` role needs OIDC provider permissions to plan the cicd module
- This was bootstrapped manually in March 2026 — if the permissions are missing again (e.g., after a role recreation), apply locally with `-target=aws_iam_role_policy.iam_role_management` to unblock CI
