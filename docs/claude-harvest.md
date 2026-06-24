# `claude-harvest.yml` — Upstream Pattern Harvest

The inverse of [`claude-audit.yml`](claude-audit.md): instead of pushing a checklist *down*
into client repos, it harvests generalizable patterns *up* from client repos into this
template/reference repo (e.g. `airflow3-best-practice`). Runs on a schedule (or on demand),
opens **one PR per harvested pattern**, and scores **cross-client corroboration** as a
strong signal.

The detection/generalization intelligence lives in the `gemma-upstream-harvest` plugin
(`gemma-agentic-toolkit`); this workflow orchestrates it.

---

## Architecture

```
Template repo (caller)        Gemma-Analytics/.github                 Client repos (multi-org)
──────────────────────        ───────────────────────                ────────────────────────
.github/workflows/
  harvest.yml
    on: schedule (monthly)    claude-harvest.yml (reusable)
    on: workflow_dispatch     ┌──────────────────────────────────────────────────────────────┐
      {pr_url?}               │ JOB 1 — detect (read-only)                                    │
    └── workflow_call ───────▶│  • checkout template (the baseline)                           │
                              │  • App token (Gemma-Analytics) → checkout harvest skills      │
                              │  • mint per-org App tokens (JWT) ───────────────────────────▶ clone sources
                              │  • collect label-nominated PR diffs ────────────────────────▶ (read-only)
                              │  • gather already-attempted ids (open+closed harvest PRs)     │
                              │  • OIDC → Bedrock                                             │
                              │  • run detect-upstream-candidates → candidates.json           │
                              │       (denylist · discriminator · corroboration · dedup)      │
                              │  • outputs: ids[], count; uploads candidates.json artifact    │
                              ├──────────────────────────────────────────────────────────────┤
                              │ JOB 2 — harvest (matrix over ids, if count > 0)               │
                              │  for each candidate id:                                       │
                              │  • checkout template; App token (this repo) for PR            │
                              │  • download candidates.json                                   │
                              │  • run generalize-pattern → applies change + ledger + PR body │
                              │  • GATE 1: diff-grep leak guard (client org / secret / deny)  │
                              │  • peter-evans/create-pull-request → branch harvest/<id>      │
                              │  • post session cost                                          │
                              └──────────────────────────────────────────────────────────────┘

Decline loop: harvest-decline-handler.yml (separate caller, on PR close) records merged
harvest PRs as `ported` and closed-unmerged ones as `declined` in the ledger.
```

### Modes
- **`scan`** (default): clone each source, diff eligible files vs the template, AND collect
  every source PR labeled `upstream-candidate` (the nomination path). Cross-client
  corroboration is scored across all sources.
- **`pr`**: harvest a single nominated client PR passed as `pr_url` — the manual override /
  smoke-test path.

### Cross-client corroboration
When 2+ clients independently make the same directional change relative to the template, the
candidate is marked `corroborated`, sorted first, and may be promoted even if borderline.
The PR body shows only the aggregate ("🔁 Corroborated by N independent client deployments")
— never client names (see Safety).

---

## Client setup (template repo)

You need **two** thin wrappers plus a ledger and a label.

### 1. The harvest caller — `.github/workflows/harvest.yml`

```yaml
name: Upstream Harvest

on:
  schedule:
    - cron: "0 7 1 * *"   # 07:00 UTC on the 1st (offset from the 06:00 audit)
  workflow_dispatch:
    inputs:
      pr_url:
        description: "Harvest a single client PR (override). Leave blank for a full scan."
        required: false

permissions:
  id-token: write
  contents: write
  pull-requests: write

jobs:
  harvest:
    uses: Gemma-Analytics/.github/.github/workflows/claude-harvest.yml@<SHA>
    with:
      sources: |
        Saxonia-Catering/data-orchestration
        Natsana/airflow-orchestration
      mode: ${{ github.event.inputs.pr_url && 'pr' || 'scan' }}
      pr_url: ${{ github.event.inputs.pr_url }}
    secrets:
      CLAUDE_CODE_ROLE_ARN: ${{ secrets.CLAUDE_CODE_ROLE_ARN }}
      GEMMA_CLAUDE_ASSISTANT_APP_ID: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_ID }}
      GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY }}
```

### 2. The decline handler — `.github/workflows/harvest-decline.yml`

```yaml
name: Harvest Decline Handler

on:
  pull_request:
    types: [closed]

permissions:
  contents: write
  pull-requests: read

jobs:
  record:
    uses: Gemma-Analytics/.github/.github/workflows/harvest-decline-handler.yml@<SHA>
    secrets:
      GEMMA_CLAUDE_ASSISTANT_APP_ID: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_ID }}
      GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY: ${{ secrets.GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY }}
```

> The handler pushes a one-line ledger update to the default branch. If that branch requires
> PRs (protection), it logs a warning instead — record the decline manually, or adapt the
> wrapper to open a PR.

### 3. The ledger — `.github/harvest-ledger.yml`

Seed it (even empty, `entries: []`) so dedup has something to read. Format and an example are
in the plugin's `harvest-ledger-template.yml`. Statuses: `proposed` | `ported` | `declined`.

> In CI, harvest PRs do **not** write `proposed` entries (parallel PRs would collide on the
> file at merge time). While a PR is open, dedup relies on the PR itself; on close/merge the
> decline-handler writes `declined`/`ported`. The ledger is thus the durable record of
> *outcomes* plus any human-curated notes — not in-flight proposals.

### 4. The label — `upstream-candidate`

Create this label in each **client** source repo. Consultants add it to a client PR to
nominate the change; the next scan collects it. (Real-time `repository_dispatch` nomination
is a planned phase 2.)

> **Cross-org note:** `secrets: inherit` does not work cross-org — pass all three secrets
> explicitly, as shown. The GitHub App must be installed on every source org with
> `contents:read` (already the case for clients using the review/audit workflows).

---

## Inputs

| Input | Default | Description |
|---|---|---|
| `sources` | _(required)_ | Client repos to harvest from, `org/repo` per line. The App must be installed on each org. |
| `mode` | `scan` | `scan` (diff sources vs template + collect nominated PRs) or `pr` (single PR via `pr_url`). |
| `pr_url` | _(empty)_ | The client PR URL to harvest when `mode: pr`. |
| `nomination_label` | `upstream-candidate` | Label that nominates a client PR (scan mode collects these). |
| `denylist` | infra default | Newline globs never read/harvested. Extends the plugin default; don't shrink it. |
| `ledger_path` | `.github/harvest-ledger.yml` | Dedup ledger location in this repo. |
| `model` | `eu.anthropic.claude-sonnet-4-6` | Bedrock model ID. |
| `aws_region` | `eu-central-1` | AWS region for Bedrock OIDC. |
| `max_turns` | `60` | Max agentic turns per Claude run. |
| `toolkit_ref` | `main` | `gemma-agentic-toolkit` ref for the skills. Pin to `gemma-upstream-harvest@1.0.0` for reproducibility. |

## Secrets (all required)

| Secret | Purpose |
|---|---|
| `CLAUDE_CODE_ROLE_ARN` | IAM role ARN for OIDC → Bedrock |
| `GEMMA_CLAUDE_ASSISTANT_APP_ID` | GitHub App ID (bot identity + cross-org reads + PR creation) |
| `GEMMA_CLAUDE_ASSISTANT_APP_PRIVATE_KEY` | GitHub App private key (PEM) |

---

## Safety model

Eligibility is broad (everything minus the denylist) and the output is a direct code PR, so
three controls carry the weight (spec: the plugin's `required-ci-gates.md`):

1. **Generalization contract** — `generalize-pattern` strips every client specific and
   documents what it removed in the PR body.
2. **Gate 1 — diff-grep leak guard** — before any PR opens, the workflow mechanically fails
   the job if the diff contains a client **org** identifier from the candidate's sources, a
   secret-shaped string, or a denylisted path. Belt-and-suspenders on the skill's self-check.
3. **Anonymized PR bodies** — aggregate corroboration counts only; precise source refs live
   only in the ledger (internal to this repo, which has a wider audience than any one client
   repo).

Plus: dedup via the ledger **and** existing `harvest/<id>` PRs (open or closed), so closed
candidates are never re-proposed; and nothing auto-merges — every change is a reviewable PR.

---

## Pinning

Scheduled, unattended runs should pin the reusable workflow to a SHA and the skills to a
release tag:

```yaml
uses: Gemma-Analytics/.github/.github/workflows/claude-harvest.yml@<SHA>
with:
  toolkit_ref: "gemma-upstream-harvest@1.0.0"
```
