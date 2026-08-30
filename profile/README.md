# x-cmd-action

GitHub Actions for the [x-cmd](https://github.com/x-cmd/x-cmd) ecosystem. Every action in this org is **pure shell** — no Node.js runtime, no bundled JS, no nested action dependencies.

[中文](./README.cn.md)

## Actions

| Action | Description | Latest |
| --- | --- | --- |
| [`x-cmd`](./x-cmd) | Install x-cmd into `~/.x-cmd.root/`. Single-purpose, idempotent, exposes `outputs.root` for chaining. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`checkout`](./checkout) | Pure-shell `git checkout`. Drop-in alternative to `actions/checkout` — 20 inputs, same surface. | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`gitmirror`](./gitmirror) | Cross-platform git repo mirror (GitHub ↔ Gitee ↔ Codeberg). Three list-source styles, fan-out concurrency. | ![v1](https://img.shields.io/badge/v1-stable-green) |

## How they fit together

```
┌─────────────────────────────────────────────────────┐
│  Your workflow                                      │
└─────────────────┬───────────────────────────────────┘
                  │
       ┌──────────┼──────────┬──────────────┐
       ▼          ▼          ▼              ▼
   x-cmd      checkout    gitmirror     x-cmd/action
   (just      (clone     (sync         (full bootstrap:
   install)   repo)      repos)         x-cmd + SSH + git
                                          + docker + ws)
```

- **`x-cmd`** = install x-cmd only. Use when you want `x` available and handle the rest yourself.
- **`checkout`** = clone a repo into the workspace. Use instead of `actions/checkout` when you don't want Node.js.
- **`gitmirror`** = periodic one-way replication across platforms. Use when you maintain a mirror on Gitee/Codeberg.
- **[`x-cmd/action`](https://github.com/x-cmd/action)** (sibling org repo) = the full CI bootstrap. Install x-cmd AND wire up SSH / git identity / docker / workspace / artifact in one step.

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