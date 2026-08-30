# x-cmd-action

GitHub Actions for the [x-cmd](https://github.com/x-cmd/x-cmd) ecosystem. Every action in this org is **pure shell** — no Node.js runtime, no bundled JS, no nested action dependencies.

[中文](./README.cn.md)

## Actions

Two layers. Layer 1 actions bring the runner close to a local dev environment; Layer 2 actions are higher-level automations built on top.

### Layer 1 — Basic Setup

Composable environment-setup actions. Each does one thing; pick what you need.

| Action | Description | Latest |
| --- | --- | --- |
| [`x-cmd`](https://github.com/x-cmd-action/x-cmd) | Install x-cmd into `~/.x-cmd.root/`. Single-purpose, idempotent. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`this-repo`](https://github.com/x-cmd-action/this-repo) | Pure-shell minimal checkout: clone trigger repo into `$GITHUB_WORKSPACE` using the runner's token. 6 inputs. Use when `checkout` is overkill. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ssh`](https://github.com/x-cmd-action/ssh) | Pure-shell `ssh-agent` setup + `known_hosts` + optional key add. Extracted from `x-cmd/action`. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`docker`](https://github.com/x-cmd-action/docker) | Pure-shell `docker login` + `docker buildx init`. Extracted from `x-cmd/action`. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`gitconfig`](https://github.com/x-cmd-action/gitconfig) | Pure-shell **global** git config. Default sets `user.name`/`user.email`; `config` input adds an `[include]` to `~/.gitconfig`. Position-independent — no `local-config` here (use `checkout`/`this-repo` for repo-scoped overlay). | ![v1](https://img.shields.io/badge/v1-stable-green) |

### Layer 2 — Common Functions

Higher-level automations. Each composes Layer 1 actions (and x-cmd commands) into a recurring workflow job.

| Action | Description | Latest |
| --- | --- | --- |
| [`checkout`](https://github.com/x-cmd-action/checkout) | Pure-shell `git checkout`. 22 inputs, 1:1 with `actions/checkout@v4` plus 3 x-cmd enhancements (`known-hosts-url`, `fetch-additional`, `local-config`). Uses `GIT_SSH_COMMAND` with temp files + hardcoded `github.com` key (mirrors `actions/checkout`). | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`gitmirror`](https://github.com/x-cmd-action/gitmirror) | Sync repos across GitHub ↔ Gitee ↔ Codeberg. Three list-source styles, fan-out concurrency. Requires x-cmd (uses `x gitb backup`). | ![v1](https://img.shields.io/badge/v1-stable-green) |

More planned (`ghwatch`, `ghissuereply`, `ghissuegold`, `webmonitor`, `hnmonitor`) — see the [internal roadmap](https://github.com/x-cmd-action/.github/blob/main/README.md).

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

- **`x-cmd`** — install x-cmd. Use when you want `x` available.
- **`this-repo`** — minimal clone to `$GITHUB_WORKSPACE`. Use when `checkout` is overkill.
- **`checkout`** — full `actions/checkout` parity (SSH, sparse, filter, fetch-additional, known-hosts-url, gitconfig). Use when you need those.
- **`gitmirror`** — periodic one-way replication across platforms.

These are peers — pick whichever you need, compose freely.

[`x-cmd/action`](https://github.com/x-cmd/action) lives in a separate repo and is a **different tool**: it installs x-cmd AND assumes you'll do the rest of the CI with x-cmd commands (`x gitb`, `x ws`, etc.) — i.e., "1 action + x-cmd commands = full CI". Use it when your whole CI is x-cmd-driven.

## Conventions

- **Repo naming:** `x-cmd-action/<name>` for actions in this org.
- **One action per repo.** No monorepo.
- **Pure shell only.** No TypeScript, no JS bundles, no `actions/setup-*` for toolchains.
- **v1+ tags available** for production; `@main` for bleeding edge.
- **Apache 2.0** across the org.
- **PRs welcome** for new actions or improvements to existing ones.

## Adding a new action

1. `gh repo create x-cmd-action/<name> --public`
2. Write `action.yml` + (optionally) `lib/<script>.sh`
3. Add the row to the table above in this `profile/README.md`
4. Submit a PR

## Related

- [x-cmd/action](https://github.com/x-cmd/action) — the full CI bootstrap (sibling, not in this org).
- [x-cmd](https://github.com/x-cmd/x-cmd) — the underlying shell library.
- [x-cmd/get](https://github.com/x-cmd/get) — x-cmd installer.