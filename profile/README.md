# x-cmd-action

GitHub Actions that bring the [x-cmd shell library](https://github.com/x-cmd/x-cmd) (and the AI toolkit wrapped by it) into CI runners. Pure shell — no Node.js, no bundled JS, no nested action dependencies.

Two reasons this org exists:

1. **Wire up the runner like a local dev box.** Install x-cmd, configure git/ssh/docker, clone the repo. Composable, pick what you need.
2. **Hand GitHub-native artifacts (issues, PRs, comments, diffs) to an LLM.** Auto-label new issues, post a draft PR review on every push, generate a weekly changelog, extract post-mortems from closed bugs. The AI is not a demo — it's the reason most users adopt this org.

[中文](./README.cn.md)

## Layer 1 — Wire up the runner like a laptop

| Action | Description | Latest |
| --- | --- | --- |
| [`x-cmd`](https://github.com/x-cmd-action/x-cmd) | Install x-cmd into `~/.x-cmd.root/`. Single-purpose, idempotent. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`this-repo`](https://github.com/x-cmd-action/this-repo) | Pure-shell minimal checkout: clone trigger repo into `$GITHUB_WORKSPACE`. 6 inputs. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ssh`](https://github.com/x-cmd-action/ssh) | Pure-shell `ssh-agent` setup + `known_hosts` + key add. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`docker`](https://github.com/x-cmd-action/docker) | Pure-shell `docker login` + `buildx init`. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`gitconfig`](https://github.com/x-cmd-action/gitconfig) | Pure-shell **global** git config (name/email + `[include]` for config file). | ![v1](https://img.shields.io/badge/v1-stable-green) |

## Layer 2 — Common CI jobs

| Action | Description | Latest |
| --- | --- | --- |
| [`checkout`](https://github.com/x-cmd-action/checkout) | Pure-shell `git checkout`. 22 inputs, 1:1 with `actions/checkout@v4` + 3 x-cmd enhancements. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`gitmirror`](https://github.com/x-cmd-action/gitmirror) | Sync repos across GitHub ↔ Gitee ↔ Codeberg. | ![v1](https://img.shields.io/badge/v1-stable-green) |

## Layer 3 — AI toolkit (LLM via x-cmd's `x ai request`)

Each Layer 3 action reads a GitHub artifact via `x gh`, asks `x ai request`, writes back via `x gh`. No AI SDK, no Node dep — the entire AI call is one `x ai request` line.

| Action | What it does for you | Latest |
| --- | --- | --- |
| [`ai/triage`](https://github.com/x-cmd-action/ai/tree/main/triage) | **Auto-label new issues.** Reads body + existing labels, AI returns `type / priority / area / labels / summary`, bot posts summary and applies labels. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/reply`](https://github.com/x-cmd-action/ai/tree/main/reply) | **First-line responder.** Watches for `@x` in issue/comment bodies, posts a reaction + reply. Combined with a FAQ, the bot cites the FAQ section that answers. **No AI token required.** | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/review`](https://github.com/x-cmd-action/ai/tree/main/review) | **PR review on every push.** Fetches the PR diff via `gh pr diff`, AI returns Security / Style / Suggestions / Summary, bot posts a structured comment. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/changelog`](https://github.com/x-cmd-action/ai/tree/main/changelog) | **Weekly community digest.** `schedule: cron`. Collects issues closed + PRs merged in the last N days, AI groups into `Features / Fixes / Performance / Docs / Other`, writes `CHANGELOG.md`. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/translate`](https://github.com/x-cmd-action/ai/tree/main/translate) | **i18n on demand.** Reads a Markdown file, translates to a target language. Markdown-aware — code blocks preserved. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/spec`](https://github.com/x-cmd-action/ai/tree/main/spec) | **RFC + post-mortem drafts.** `mode: rfc` or `mode: postmortem`. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/commit`](https://github.com/x-cmd-action/ai/tree/main/commit) | **Conventional Commits enforcement.** `mode: check` validates PR commits; `mode: generate` writes a compliant message. Pairs with `ai/changelog`. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`mneme`](https://github.com/x-cmd-action/mneme) | **AI memory layer.** Persists + retrieves LLM context across workflow runs. Lets `ai/review` on PR #123 remember what `ai/triage` said on issue #42. | ![v1](https://img.shields.io/badge/v1-stable-green) |

The full Layer 3 internals (per-action design notes, roadmap, story archive) live in the private [`mneme`](https://github.com/x-cmd-action/mneme) repo.

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
   │          checkout     ← Layer 2: full actions/checkout parity
   │          (clone+more)
   ▼
 ssh, docker, gitconfig
 (each adds one piece)
```

For Layer 3 (AI):

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
   x ai request     x mneme store/retrieve
   (call LLM)      (cross-run memory)
        │                   │
        └─────────┬─────────┘
                  ▼
            comment / labels / CHANGELOG.md
```

- **`x-cmd`** — install x-cmd. Use when you want `x` available.
- **`this-repo`** — minimal clone to `$GITHUB_WORKSPACE`. Use when `checkout` is overkill.
- **`checkout`** — full `actions/checkout` parity (SSH, sparse, filter, fetch-additional, known-hosts-url, gitconfig). Use when you need those.
- **`gitmirror`** — periodic one-way replication across platforms.
- **`ai/*`** — auto-label, first-line reply, PR review, weekly changelog, i18n, RFC drafts, commit enforcement. Compose with `mneme` for cross-run memory.

These are peers — pick whichever you need, compose freely.

[`x-cmd/action`](https://github.com/x-cmd/action) lives in a separate repo and is a **different tool**: it installs x-cmd AND assumes you'll do the rest of the CI with x-cmd commands (`x gitb`, `x ws`, etc.) — i.e., "1 action + x-cmd commands = full CI". Use it when your whole CI is x-cmd-driven.

## Conventions

- **Repo naming:** `x-cmd-action/<name>` for actions in this org. Layer 3 sub-commands live as subpaths under `x-cmd-action/ai` and are referenced as `x-cmd-action/ai/<subcmd>`.
- **One action per repo.** Layer 3 is the only exception: `x-cmd-action/ai` is a monorepo with seven sub-commands, because they share `x ai request` and `x gh` glue.
- **Pure shell only.** No TypeScript, no JS bundles, no `actions/setup-*` for toolchains.
- **v1+ tags available** for production; `@main` for bleeding edge.
- **Apache 2.0** across the org.
- **PRs welcome** for new actions or improvements to existing ones.

## Adding a new action

1. `gh repo create x-cmd-action/<name> --public`
2. Write `action.yml` + (optionally) `lib/<script>.sh`
3. Add the row to the table above in this `profile/README.md`
4. Submit a PR

For a new Layer 3 sub-command, instead add it to `x-cmd-action/ai/<subname>/` and link it from the Layer 3 table.

## Related

- [x-cmd/action](https://github.com/x-cmd/action) — the full CI bootstrap (sibling, not in this org).
- [x-cmd](https://github.com/x-cmd/x-cmd) — the underlying shell library.
- [x-cmd/get](https://github.com/x-cmd/get) — x-cmd installer.
- [mneme](https://github.com/x-cmd-action/mneme) — private org-wide memory + design notes (Layer 3 internals live here).