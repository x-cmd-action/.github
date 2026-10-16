# x-cmd-action

A collection of **pure-shell** GitHub Actions built on the [x-cmd shell library](https://github.com/x-cmd/x-cmd): set up the runner (install x-cmd, clone the repo, configure git/ssh/docker), and let AI take over the repetitive maintenance work on Issues & PRs. No Node.js dependencies — every action is one `uses:` line.

[中文](./README.cn.md)

## What's here

### Runner setup — install x-cmd, clone the repo, configure git / ssh / docker

| Action | Description | Latest |
| --- | --- | --- |
| [`x-cmd`](https://github.com/x-cmd-action/x-cmd) | Install x-cmd into `~/.x-cmd.root/` so `x ai`, `x gh`, etc. are available. Single-purpose, idempotent. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`this-repo`](https://github.com/x-cmd-action/this-repo) | Minimal checkout: clone the triggering repo into `$GITHUB_WORKSPACE`. 6 inputs. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ssh`](https://github.com/x-cmd-action/ssh) | Pure-shell `ssh-agent` setup + `known_hosts` + key add. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`docker`](https://github.com/x-cmd-action/docker) | Pure-shell `docker login` + `buildx init`. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`gitconfig`](https://github.com/x-cmd-action/gitconfig) | Pure-shell **global** git config (name/email + `[include]` for config file). | ![v1](https://img.shields.io/badge/v1-stable-green) |

### Common CI jobs

| Action | Description | Latest |
| --- | --- | --- |
| [`checkout`](https://github.com/x-cmd-action/checkout) | Full `git checkout`: 22 inputs, 1:1 with `actions/checkout@v4` + 3 x-cmd enhancements (SSH, sparse, filter, …). | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`gitmirror`](https://github.com/x-cmd-action/gitmirror) | Sync repos across GitHub ↔ Gitee ↔ Codeberg. | ![v1](https://img.shields.io/badge/v1-stable-green) |

### AI toolkit — automated Issue & PR maintenance

Auto-label new issues, auto-review PRs, auto-generate weekly changelogs. The token is read from env vars (`MINIMAX_API_KEY`, `DEEPSEEK_API_KEY`, `OPENAI_API_KEY`, …) — set one repo secret and you're done. **The AI drafts; humans decide.**

| Action | What it does for you | Latest |
| --- | --- | --- |
| [`ai/triage`](https://github.com/x-cmd-action/ai/tree/main/triage) | **Auto-label new issues.** Reads body + existing labels, AI returns `type / priority / area / labels / summary`, bot posts summary and applies labels. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/reply`](https://github.com/x-cmd-action/ai/tree/main/reply) | **First-line responder.** Watches for `@x` in issue/comment bodies, posts a reaction + reply. Static reply by default; set `use_ai: 'true'` to generate the reply from context. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/review`](https://github.com/x-cmd-action/ai/tree/main/review) | **PR review on every push.** Fetches the PR diff via `gh pr diff`, AI returns Security / Style / Suggestions / Summary, bot posts a structured comment. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/changelog`](https://github.com/x-cmd-action/ai/tree/main/changelog) | **Weekly community digest.** `schedule: cron`. Collects issues closed + PRs merged in the last N days, AI groups into `Features / Fixes / Performance / Docs / Other`, writes `CHANGELOG.md`. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/translate`](https://github.com/x-cmd-action/ai/tree/main/translate) | **i18n on demand.** Reads a Markdown file, translates to a target language. Markdown-aware — code blocks preserved. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/spec`](https://github.com/x-cmd-action/ai/tree/main/spec) | **RFC + post-mortem drafts.** `mode: rfc` or `mode: postmortem`. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/commit`](https://github.com/x-cmd-action/ai/tree/main/commit) | **Conventional Commits enforcement.** `mode: check` validates PR commits; `mode: generate` writes a compliant message. Pairs with `ai/changelog`. | ![v1](https://img.shields.io/badge/v1-stable-green) |

Cross-run memory (so `ai/review` can recall what `ai/triage` said on the linked issue) is handled internally — users get useful drafts without wiring it up.

## Quick examples

Set up the basics (x-cmd, repo, git identity):

```yaml
steps:
  - uses: x-cmd-action/x-cmd@v1
  - uses: x-cmd-action/this-repo@v1
  - uses: x-cmd-action/gitconfig@v1
    with:
      email: bot@example.com
```

Auto-label a new issue (add `MINIMAX_API_KEY` to repo secrets first; `DEEPSEEK_API_KEY` / `OPENAI_API_KEY` etc. also work):

```yaml
on:
  issues:
    types: [opened]
jobs:
  triage:
    runs-on: ubuntu-latest
    permissions: { contents: read, issues: write }
    steps:
      - uses: x-cmd-action/ai/triage@v1
        with: { apply-labels: 'true' }
        env: { MINIMAX_API_KEY: ${{ secrets.MINIMAX_API_KEY }} }
```

First-line responder with an AI-generated reply on `@x` (drop `use_ai` and `env` for a static reply — no AI token needed):

```yaml
on:
  issue_comment:
    types: [created]
concurrency:
  group: aireply
  cancel-in-progress: false
jobs:
  reply:
    if: github.event.sender.type != 'Bot'
    runs-on: ubuntu-latest
    permissions: { contents: read, issues: write }
    steps:
      - uses: x-cmd-action/ai/reply@v1
        with:
          keyword: '@x'
          reaction: eyes
          use_ai: 'true'           # generate the reply from context
        env:
          MINIMAX_API_KEY: ${{ secrets.MINIMAX_API_KEY }}
```

## Design principles

1. **Pure shell** — no Node.js, no bundled JS, no nested action dependencies. Avoids the Node/Python/library-version churn that forces frequent action version bumps.
2. **Recurring ops live in portable shell / Node.js / Python scripts.** The actions wire up the runner — install x-cmd, configure git/ssh/docker, clone the repo. The actual workflow logic lives in `x gitb backup`, `x gh`, `x ws`, … — scripts that run on your laptop, on a cron, on any CI. When the workflow moves off GitHub, the work moves with it.
3. **Safe, lightweight AI assist for maintainers.** Auto-label new issues, draft a PR review on every push, generate a weekly changelog, extract post-mortems from closed bugs. **The AI drafts; humans decide.** Every AI output is a starting point a maintainer reviews and edits, not an autonomous agent pushing to `main`.

## How they fit together

```
┌─────────────────────────────────────────────────────┐
│  Your workflow                                      │
└─────────────────┬───────────────────────────────────┘
                  │
   ┌──────────────┼──────────────┐
   ▼              ▼              ▼
 x-cmd        this-repo      gitmirror
 (install)    (clone this)   (sync)
   │              │
   │              ▼
   │          checkout     ← full actions/checkout parity
   │          (clone+more)
   ▼
 ssh, docker, gitconfig
 (each adds one piece)
```

For the AI toolkit:

```
┌─────────────────────────────────────────────────────┐
│  Issue opened / PR opened / cron weekly             │
└─────────────────┬───────────────────────────────────┘
                  │
                  ▼
            ai/triage / ai/review / ai/changelog / ...
                  │
        ┌─────────┼─────────┐
        ▼                   ▼
   x ai request     cross-run memory
   (call LLM)      (handled internally)
        │                   │
        └─────────┬─────────┘
                  ▼
            comment / labels / CHANGELOG.md
```

- **`x-cmd`** — install x-cmd. Use when you want `x` available.
- **`this-repo`** — minimal clone to `$GITHUB_WORKSPACE`. Use when `checkout` is overkill.
- **`checkout`** — full `actions/checkout` parity (SSH, sparse, filter, fetch-additional, known-hosts-url, gitconfig). Use when you need those.
- **`gitmirror`** — periodic one-way replication across platforms.
- **`ai/*`** — auto-label, first-line reply, PR review, weekly changelog, i18n, RFC drafts, commit enforcement. Cross-run memory is handled internally; users don't wire it up.

These are peers — pick whichever you need, compose freely.

[`x-cmd/action`](https://github.com/x-cmd/action) lives in a separate repo and is a **different tool**: it installs x-cmd AND assumes you'll do the rest of the CI with x-cmd commands (`x gitb`, `x ws`, etc.) — i.e., "1 action + x-cmd commands = full CI". Use it when your whole CI is x-cmd-driven.

## Conventions

- **Repo naming:** `x-cmd-action/<name>` for actions in this org. AI sub-commands live as subpaths under `x-cmd-action/ai` and are referenced as `x-cmd-action/ai/<subcmd>`.
- **One action per repo.** The AI toolkit is the only exception: `x-cmd-action/ai` is a monorepo with seven sub-commands, because they share the `x ai` and `x gh` glue.
- **Pure shell only.** No TypeScript, no JS bundles, no `actions/setup-*` for toolchains.
- **v1+ tags available** for production; `@main` for bleeding edge.
- **Apache 2.0** across the org.
- **PRs welcome** for new actions or improvements to existing ones.

## Adding a new action

1. `gh repo create x-cmd-action/<name> --public`
2. Write `action.yml` + (optionally) `lib/<script>.sh`
3. Add the row to the table above in this `profile/README.md`
4. Submit a PR

For a new AI sub-command, instead add it to `x-cmd-action/ai/<subname>/` and link it from the AI toolkit table.

## FAQ

### How do I configure the AI token? Which providers are supported?

Every AI-calling sub-command reads its token from an environment variable — pass it via `env:` + secrets, e.g. `MINIMAX_API_KEY: ${{ secrets.MINIMAX_API_KEY }}`. x-cmd detects the provider from the key — supported: `MINIMAX_API_KEY`, `DEEPSEEK_API_KEY`, `GEMINI_API_KEY`, `XAI_API_KEY`, `MISTRAL_API_KEY`, `MOONSHOT_API_KEY`, `OPENAI_API_KEY`, etc. Exceptions: `ai/reply` with a static comment and `ai/commit` in `check` mode (format validation only) don't need a token.

### Will the AI modify my repo or push to main on its own?

No. All AI output is a **draft**: a comment on the issue/PR, suggested labels, a file written to the workspace. No action commits or pushes autonomously — a human decides what lands.

### Can I use this outside GitHub Actions?

The actions are GitHub Actions wrappers, but the core logic is x-cmd commands: with x-cmd installed locally, `x ai triage`, `x ai review`, etc. run directly. Tune prompts and behavior locally before wiring up CI.

### What if a label suggested by ai/triage doesn't exist in the repo?

It's created automatically before being applied — no need to pre-create labels.

### Why shell / Node.js / Python for workflow logic, instead of composing `action.yml`?

Recurring ops work lives in portable shell / Python / JS scripts (`x gitb backup`, `x gh`, `x ws`, …) — scripts you can run on your laptop, on a cron, on any CI. These actions are **thin wrappers around those scripts**, not opaque GitHub-Actions-only logic. When the workflow moves off GitHub, the work moves with it.

## Related

- [x-cmd/action](https://github.com/x-cmd/action) — the full CI bootstrap (sibling, not in this org).
- [x-cmd](https://github.com/x-cmd/x-cmd) — the underlying shell library.
- [x-cmd/get](https://github.com/x-cmd/get) — x-cmd installer.
- (no public cross-run memory action — handled inside the AI toolkit)
