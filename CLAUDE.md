# Claude Code — Contributor Guidelines

This file is read automatically by Claude Code during code review and agentic tasks in this repository.

## Repository purpose

This is the `Gemma-Analytics/.github` central workflow repository. It hosts reusable GitHub Actions workflows that are consumed by client repos across the Gemma Analytics organisation. Changes here can affect every client repo simultaneously.

## Adding a new workflow

Every new reusable workflow **must** be accompanied by:

1. **A documentation file** at `docs/<workflow-name>.md` covering:
   - What the workflow does and when it fires
   - An ASCII architecture diagram showing the trigger → steps → outputs flow
   - A copy-paste client wrapper snippet with all inputs and secrets shown explicitly
   - A table of all `inputs` with types, defaults, and a description of when to change each
   - Any security or operational notes relevant to the workflow

2. **A link in `README.md`** — add a row to the Workflows table with the workflow name, a one-line description, and a link to the new doc file.

PRs that add a workflow without the matching doc file should be treated as incomplete.

## Modifying an existing workflow

- If you change the default value of an input, update the corresponding table row in `docs/<workflow-name>.md`.
- If you add or remove an input, update both the inputs table and the client wrapper snippet in `docs/<workflow-name>.md`.
- If you change the trigger events or concurrency behaviour, update the architecture diagram in `docs/<workflow-name>.md`.

## Security rules

- Never hardcode secrets, tokens, or credentials in any workflow file.
- User-supplied values (comment bodies, PR titles, issue text) must always be passed via `env:` variables in `run:` steps — never interpolated directly into shell commands.
- The `allowed_bots` setting must include `gemma-claude-assistant[bot]` and `gemmbot` to prevent bot-triggered loops.
- Workflows that create PRs or post comments must use the GitHub App token (`steps.app-token.outputs.token`), not `GITHUB_TOKEN`, so actions appear under the bot identity.

## Cross-org considerations

- `secrets: inherit` does not work across organisations. Client wrappers in non-Gemma-Analytics orgs must pass all secrets explicitly.
- When checking out a private repo from a different org, a separate App token scoped to that org is required (see `claude-audit.yml` for the two-token pattern).
