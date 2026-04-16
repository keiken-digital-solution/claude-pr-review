# Claude PR Review

A shared, reusable GitHub Actions workflow that runs a Claude-powered code review on every pull request. Consumers call it from their own repositories and pick a base prompt tailored to their code type (application vs. infrastructure), with hooks to override or extend the prompt and to escalate/de-escalate the model per PR.

---

## Overview

This repository hosts:

- A **reusable workflow** (`.github/workflows/review.yml`) that consumer repositories invoke on pull request events.
- Two **base prompts** (`prompts/codebase-review.md`, `prompts/infra-review.md`) that Claude follows when reviewing.
- Documentation on how to wire it up, customize it, and run it reliably.

Claude reviews only the **diff of the PR**, but has access to the **full repository** and is instructed to load any `CLAUDE.md` or architecture doc before starting, so it reviews with context, not in isolation.

## Quick start

In any consumer repository, add `.github/workflows/claude-review.yml`:

```yaml
name: Claude Review

on:
  pull_request:
    types: [opened, synchronize, reopened, labeled, unlabeled]

jobs:
  review:
    uses: YOUR-ORG/claude-pr-review/.github/workflows/review.yml@v1
    with:
      prompt-type: codebase-review
      shared-repo: YOUR-ORG/claude-pr-review
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      # OR, if you use Claude subscription auth:
      # CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

Add **one** of the following secrets in the consumer repository's Settings → Secrets and variables → Actions:

- `ANTHROPIC_API_KEY` — an Anthropic API key.
- `CLAUDE_CODE_OAUTH_TOKEN` — a Claude Code OAuth token (if your org uses Claude subscription auth rather than direct API billing).

The secret name you set is the same name you reference in `secrets:` when calling the reusable workflow (see examples below).

## Inputs

| Input | Default | Description |
|---|---|---|
| `prompt-type` | `""` | Base prompt identifier: `codebase-review` or `infra-review`. Required unless `custom-prompt-path` is set. |
| `custom-prompt-path` | `""` | Path **in the consumer repo** to a prompt file that **replaces** the base prompt entirely. |
| `append-prompt` | `""` | Text **appended** to the end of the prompt (base or custom). Use this to layer team-specific instructions without replacing the base. |
| `claude-model` | `claude-sonnet-4-5` | Default model used for reviews. |
| `deep-model` | `claude-opus-4-6` | Model used when `deep-triggers` match or `deep-label` is set. |
| `quick-model` | `claude-haiku-4-5` | Model used when `quick-label` is set. |
| `deep-triggers` | `""` | Comma-separated extended-regex patterns. If **any** changed file path matches, the review escalates to `deep-model`. Example: `^auth/,terraform/prod/,^billing/`. |
| `deep-label` | `deep-review` | PR label name that forces `deep-model`. |
| `quick-label` | `quick-review` | PR label name that forces `quick-model`. |
| `shared-repo` | `YOUR-ORG/claude-pr-review` | `owner/repo` of this shared workflow repository. |
| `shared-repo-ref` | `main` | Git ref (branch, tag, or SHA) of the shared repo for prompt fetching. |

### Secrets

At least one of the following must be provided:

| Secret | Description |
|---|---|
| `ANTHROPIC_API_KEY` | Anthropic API key. |
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude Code OAuth token (Claude subscription auth). |

## Model resolution

When a review runs, the model is selected using this priority order (first match wins):

1. **`deep-triggers`** — if any changed file path matches any of the regex patterns, the review uses `deep-model`.
2. **`deep-label`** — if the PR has this label, the review uses `deep-model`.
3. **`quick-label`** — if the PR has this label, the review uses `quick-model`.
4. **`claude-model`** — the default model.

This gives you three levers with different ergonomics:

- **`deep-triggers`** is automatic and always-on. Use it for paths where deep review is non-negotiable (e.g. `auth/`, `billing/`, production IaC).
- **`deep-label`** / **`quick-label`** are self-service. An author who knows their PR warrants more or less scrutiny can add the label and re-trigger the review.
- **`claude-model`** is the per-repo baseline.

## Customization: override vs. append

Two independent mechanisms, documented side by side so teams can choose:

### `custom-prompt-path` — override

Replaces the base prompt **entirely**. The file at the given path in the consumer repo becomes the sole prompt.

```yaml
with:
  custom-prompt-path: .github/review-prompt.md
```

Use this when the base prompt does not fit your team's review style at all.

**Caveat:** you no longer benefit from updates to the base prompt in this shared repo; you maintain your own.

### `append-prompt` — append

Appends extra instructions to the end of the prompt (base or custom).

```yaml
with:
  prompt-type: codebase-review
  append-prompt: |
    Additional team-specific rules:
    - All database access must go through our `repositories/` layer, never direct ORM calls from handlers.
    - Use the internal `logger` module (not `console.log` / `print`).
    - Every new public API must be documented in `docs/api/`.
```

Use this to layer team-specific rules on top of a maintained base prompt. Future updates to the base prompt flow through automatically.

## Recommended: maintain a `CLAUDE.md` in your repo

The base prompts instruct Claude to read `CLAUDE.md` (or any top-level architecture doc) before starting the review. This is how Claude gets **persistent context** about your repo without re-discovering it on every PR.

Put in `CLAUDE.md` whatever a new engineer would need on day one:

- Purpose and scope of the repo.
- Tech stack and key dependencies.
- Architectural layout (modules, packages, layers).
- Conventions (naming, error handling, logging, testing patterns).
- Protected zones ("never touch X", "prefer Y over Z").
- Common commands (how to run tests, lint, format).

A good `CLAUDE.md` dramatically improves review quality because Claude starts from a brief instead of exploring blindly.

## Examples

### Application repository (e.g. `keiken-dragon-ai`)

```yaml
name: Claude Review

on:
  pull_request:
    types: [opened, synchronize, reopened, labeled, unlabeled]

jobs:
  review:
    uses: YOUR-ORG/claude-pr-review/.github/workflows/review.yml@v1
    with:
      prompt-type: codebase-review
      shared-repo: YOUR-ORG/claude-pr-review
      deep-triggers: "^auth/,^billing/,^services/payments/"
      append-prompt: |
        Team-specific rules:
        - No direct database calls outside `repositories/`.
        - All HTTP handlers must validate input via the `pydantic` schemas in `schemas/`.
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

### Infrastructure repository (e.g. `kpp-platform-infra`)

```yaml
name: Claude Review

on:
  pull_request:
    types: [opened, synchronize, reopened, labeled, unlabeled]

jobs:
  review:
    uses: YOUR-ORG/claude-pr-review/.github/workflows/review.yml@v1
    with:
      prompt-type: infra-review
      shared-repo: YOUR-ORG/claude-pr-review
      deep-triggers: "terraform/prod/,^modules/networking/,^modules/iam/"
      append-prompt: |
        Team-specific rules:
        - Any new S3 bucket must have `force_destroy = false` and a lifecycle policy.
        - Production changes require a reference to a change ticket in the PR description.
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

## Versioning

Pin the workflow to a tag for stability:

```yaml
uses: YOUR-ORG/claude-pr-review/.github/workflows/review.yml@v1
```

Available refs:

- `@v1` — stable major version, recommended for consumer repos.
- `@main` — latest changes, may break without notice.
- `@<sha>` — a specific commit, for reproducible setups.

When pinning to `@v1`, make sure `shared-repo-ref` input also points to `v1` (or leave it at `main` if you want the latest prompts regardless of workflow version — they evolve independently).

## What you will see as a PR author

When a PR triggers the workflow, the Claude action automatically:

- Adds a 👀 reaction on the PR.
- Posts a sticky comment (`🤖 Claude is reviewing…`) that **updates in real time** as Claude explores the repo and produces findings.
- Creates a GitHub Actions check visible in the PR's **Checks** section (status: in progress → success / failure).
- Links back to the workflow run for full logs.

No extra setup needed on the consumer side; this is native behavior of `anthropics/claude-code-action`.

## Troubleshooting

**No authentication provided** — add either `ANTHROPIC_API_KEY` or `CLAUDE_CODE_OAUTH_TOKEN` under Settings → Secrets and variables → Actions in the consumer repository, and reference it in the `secrets:` block of the caller workflow.

**`custom-prompt-path 'X' was not found`** — the path is resolved relative to the root of the consumer repository. Make sure the file is committed on the PR branch.

**`prompt-type must be 'codebase-review' or 'infra-review'`** — either pass a valid `prompt-type`, or pass a `custom-prompt-path`.

**Shared prompts not fetching** — verify the `shared-repo` input points at the right `owner/repo`. If this shared repo is private, the consumer workflow needs a token with read access (pass via a PAT or GitHub App token — see GitHub docs on cross-repository checkout).

**Action inputs changed** — the `anthropics/claude-code-action` action may evolve. If a run fails at the last step with an unknown-input error, check the action's release notes and update `.github/workflows/review.yml` accordingly.

## License

MIT. See [LICENSE](LICENSE).
