# sidehustle-ci

Shared **reusable GitHub Actions workflows** for the sidehustle fleet.

This repo is **public on purpose** and contains **only workflow YAML** — no product code, and
no secret *values* (only secret *names*). GitHub Free/Pro/Team cannot lend a *private* reusable
workflow to *other private* repos (only same-repo or public; cross-private needs Enterprise).
Putting the shared workflow in a public, YAML-only repo solves that without exposing anything:
the private caller repos inject their secrets at runtime via `secrets: inherit`.

## What's here

- `.github/workflows/pr-gates-reusable.yml` — the full PR pipeline (`on: workflow_call`):
  `changes (paths-filter) → workflow-lint → lint, typecheck, unit → e2e → claude-review
  (Gate 2, DeepSeek) → codex-adversarial (Gate 3, OpenAI) + architect-gate (tier-2/3,
  Anthropic Opus) → gates-green` (alls-green aggregate + `gh pr merge --auto --squash`).
  Plus `eval-gate (advisory)` — runs [promptfoo](https://www.promptfoo.dev/) against an
  OPTIONAL `promptfooconfig.yaml` in the caller (root or `promptfoo/`) with a cheap DeepSeek
  judge, posts a `## Eval Gate` comment, and is a no-op PASS when the caller has no config. It
  is **not** in `gates-green`'s `needs` and **not** a required check, so it never blocks merge.

### Cost controls (v3)

- **Doc-only PRs skip the expensive jobs.** The first job, `changes`
  ([dorny/paths-filter](https://github.com/dorny/paths-filter)), classifies the diff. A PR that
  only touches docs (`**/*.md`, `specs/**`, `.claude/rules/**`, `docs/**`, `LICENSE`) sets
  `code=false`, which **skips** `e2e` and all three AI gates (`claude-review`,
  `codex-adversarial`, `architect-gate`) and the advisory `eval-gate`. `lint` / `typecheck` /
  `unit` always run as the cheap floor, so a doc-only PR still aggregates green and merges
  (`gates-green` forgives the skipped gates via alls-green `allowed-skips`).
- **Cheap before expensive.** `e2e` and the AI gates only start after `lint` + `typecheck` are
  green (they are in `needs`), so a lint/type error never burns a Playwright run or LLM quota.
- **Playwright browsers are cached** on `~/.cache/ms-playwright`, keyed on OS + the installed
  Playwright version.

### Gate 3 (codex) — tier-gated, delta-aware

`codex-adversarial` recomputes the risk tier at runtime from the diff (the caller's
`scripts/classify-tier.sh`, exactly like `architect-gate`). It **blocks only on tier-2/3** PRs
(a CRITICAL finding fails the gate); on **tier-1** it runs as **pure advisory** and never fails.
Gate 2 (`claude-review`) remains the blocking floor on every code PR regardless of tier. On a
re-run (`synchronize`) Gate 3 is **delta-aware**: it reads its previous marker-tagged verdict
comment, re-checks each prior finding against the new HEAD, and limits new findings to the
increment — instead of re-litigating the whole diff each cycle. The posted comment shows
CRITICAL findings prominently and folds ADVISORY findings into a `<details>` block.

## How a repo consumes it (thin caller)

```yaml
# .github/workflows/pr-gates.yml in each private caller repo
name: PR Gates
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review, edited, labeled, unlabeled]

concurrency:
  group: pr-gates-${{ github.event.pull_request.number }}
  cancel-in-progress: true

# REQUIRED: a called workflow's job permissions are capped by the caller's grant, and the
# repo default is read-only. Grant the union the reusable jobs need or the run fails at
# STARTUP (`startup_failure`). Each reusable job still narrows to its own least-privilege set.
permissions:
  contents: write
  pull-requests: write
  issues: write
  id-token: write

jobs:
  gates:
    uses: Jobi0202/sidehustle-ci/.github/workflows/pr-gates-reusable.yml@main
    secrets: inherit

  # Re-exports the aggregate result under the stable check name "Gates Green" that branch
  # protection requires. Reusable job checks are always prefixed "gates / ...", so keeping
  # this caller-level job means NO branch-protection edit is needed when a repo migrates.
  gates-green:
    name: Gates Green
    needs: [gates]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - run: |
          result='${{ needs.gates.result }}'
          if [ "$result" != "success" ]; then
            echo "::error::Reusable PR Gates did not succeed (result=$result)."; exit 1
          fi
          echo "Reusable PR Gates passed — Gates Green."
```

The called workflow runs in the **caller's** context: `github.event.pull_request.*` and
`github.repository` resolve to the caller's PR/repo, and each job's `actions/checkout` checks
out the **caller** repo — so `scripts/classify-tier.sh` and `.claude/rules/*` are read from the
caller (every caller is a template clone of `sidehustle-foundation`, which ships them).

### Required secrets in each caller

Set on the caller repo (values never live here): `DEEPSEEK_API_KEY`, `OPENAI_API_KEY`,
`CLAUDE_CODE_OAUTH_TOKEN`. Optional: `ANTHROPIC_API_KEY` (Gate 2 rollback),
`PUSHOVER_APP_TOKEN` / `PUSHOVER_USER_KEY` (tier-3 block alert). `GITHUB_TOKEN` is automatic.

## Pinning: `@main` (with snapshot tags `v1`/`v2`/`v3`)

Callers pin **`@main`** so a central change — e.g. the post-15.06 *Max-Switch* (Gate 3
OpenAI → Anthropic Opus) — propagates to every caller with **no child edit**. The usual
"pin an immutable SHA/tag" supply-chain caution is about untrusted third-party actions; this
repo is single-owner and YAML-only, so `@main` is the right trade-off for fleet-wide
propagation. Snapshot tags are also published for anyone who prefers an immutable-ish pin:

- **`v1`** — original monolithic-parity reusable.
- **`v2`** — adds the `merge_mode` input (`auto` | `direct`) for Free-plan hands-off merge.
- **`v3`** — cost controls (doc-only skip, cheap-before-expensive, Playwright cache) +
  advisory `eval-gate` + tier-gated, delta-aware Gate 3.

Bump the tag (or just push `main`) to roll out a central change.
