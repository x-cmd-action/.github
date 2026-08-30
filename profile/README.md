# x-cmd-action

GitHub Actions for the [x-cmd](https://github.com/x-cmd/x-cmd) ecosystem. Every action in this org is **pure shell** — no Node.js runtime, no bundled JS, no nested action dependencies.

[中文](./README.cn.md)

## Actions

Two layers. Layer 1 actions bring the runner close to a local dev environment; Layer 2 actions are higher-level automations built on top.

### Layer 1 — Basic Setup

Composable environment-setup actions. Each does one thing; pick what you need.

| Action | Description | Latest |
| --- | --- | --- |
| [`x-cmd`](./x-cmd) | Install x-cmd into `~/.x-cmd.root/`. Single-purpose, idempotent. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`checkout`](./checkout) | Pure-shell `git checkout`. 20 inputs, same surface as `actions/checkout`. **No x-cmd dep** — uses only `git`, `ssh-agent`, `ssh-keyscan`. | ![v1](https://img.shields.io/badge/v1-stable-green) |

### Layer 2 — Common Functions

Self-contained automations. Each does one recurring workflow job, built on Layer 1 + x-cmd modules.

| Action | Description | Latest |
| --- | --- | --- |
| [`gitmirror`](./gitmirror) | Sync repos across GitHub ↔ Gitee ↔ Codeberg. Three list-source styles, fan-out concurrency. Requires x-cmd (uses `x gitb backup`). | ![v1](https://img.shields.io/badge/v1-stable-green) |

More planned (`ghwatch`, `ghissuereply`, `ghissuegold`, `webmonitor`, `hnmonitor`) — see the [internal roadmap](https://github.com/x-cmd-action/.github/blob/main/README.md).

## How they fit together

```
┌─────────────────────────────────────────────────────┐
│  Your workflow                                      │
└─────────────────┬───────────────────────────────────┘
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
   x-cmd      checkout    gitmirror
   (install)  (clone)     (sync)
```

- **`x-cmd`** — install x-cmd. Use when you want `x` available.
- **`checkout`** — clone a repo into the workspace. Use instead of `actions/checkout` when you don't want Node.js.
- **`gitmirror`** — periodic one-way replication across platforms. Use when you maintain a mirror on Gitee/Codeberg.

These three are peers — pick whichever you need, compose freely.

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