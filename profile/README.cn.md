# x-cmd-action

[x-cmd](https://github.com/x-cmd/x-cmd) 生态的 GitHub Actions。这个 org 下每个 action 都是**纯 shell** —— 没有 Node.js runtime、没有 JS bundle、没有嵌套 action 依赖。

[English](./README.md)

## Actions

两层。Layer 1 让 runner 接近本地开发环境；Layer 2 是基于 Layer 1 构建的高阶自动化。

### Layer 1 — Basic Setup

可组合的环境配置 action。各做一件事，按需取用。

| Action | 说明 | 最新版 |
| --- | --- | --- |
| [`x-cmd`](https://github.com/x-cmd-action/x-cmd) | 安装 x-cmd 到 `~/.x-cmd.root/`。单一职责、幂等。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`this-repo`](https://github.com/x-cmd-action/this-repo) | 纯 shell 最精简 checkout：把触发 repo 克隆到 `$GITHUB_WORKSPACE`，用 runner 的 token。6 个 input。`checkout` 太重时用它。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ssh`](https://github.com/x-cmd-action/ssh) | 纯 shell `ssh-agent` setup + `known_hosts` + 可选 key add。从 `x-cmd/action` 抽出。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`docker`](https://github.com/x-cmd-action/docker) | 纯 shell `docker login` + `docker buildx init`。从 `x-cmd/action` 抽出。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`gitconfig`](https://github.com/x-cmd-action/gitconfig) | 纯 shell **全局** git config。默认设 `user.name`/`user.email`；`config` input 给 `~/.gitconfig` 加 `[include]`。位置无关 —— 不提供 `local-config`（repo-scoped overlay 用 `checkout` / `this-repo`）。 | ![v1](https://img.shields.io/badge/v1-stable-green) |

### Layer 2 — Common Functions

更高阶的自动化。每个把 Layer 1 action（和 x-cmd 命令）组合成一个常用 workflow 任务。

| Action | 说明 | 最新版 |
| --- | --- | --- |
| [`checkout`](https://github.com/x-cmd-action/checkout) | 纯 shell `git checkout`。22 个 input，与 `actions/checkout@v4` 1:1 对齐 + 3 个 x-cmd 增强（`known-hosts-url`、`fetch-additional`、`local-config`）。用 `GIT_SSH_COMMAND` + temp 文件 + 硬编码 `github.com` 公钥（学 `actions/checkout`）。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`gitmirror`](https://github.com/x-cmd-action/gitmirror) | 跨平台同步 repo（GitHub ↔ Gitee ↔ Codeberg）。三种 list 来源、扇出并发。需要 x-cmd（用 `x gitb backup`）。 | ![v1](https://img.shields.io/badge/v1-stable-green) |

更多计划中的 action（`ghwatch`、`ghissuereply`、`ghissuegold`、`webmonitor`、`hnmonitor`）—— 见[内部路线图](https://github.com/x-cmd-action/.github/blob/main/README.md)。

## 它们怎么配合

```
┌─────────────────────────────────────────────────────┐
│  你的 workflow                                       │
└─────────────────┬───────────────────────────────────┘
                  │
   ┌──────────────┼──────────────┐
   ▼              ▼              ▼
 x-cmd        this-repo      gitmirror
 (装)         (clone 这个)   (同步)
   │              │
   │              ▼
   │          checkout     ← Layer 2：完整 actions/checkout 对齐
   │          (clone+more)
   ▼
 ssh, docker, gitconfig
 (各加一项)
```

- **`x-cmd`** —— 装 x-cmd。想要 `x` 可用时用。
- **`this-repo`** —— 最精简 clone 到 `$GITHUB_WORKSPACE`。`checkout` 太重时用它。
- **`checkout`** —— 完整 `actions/checkout` 对齐的 clone（SSH、sparse、filter、fetch-additional、known-hosts-url、gitconfig）。需要这些时用这个。
- **`gitmirror`** —— 跨平台单向复制。

这些是同级 —— 需要哪个就拿哪个，可以自由组合。

[`x-cmd/action`](https://github.com/x-cmd/action) 在独立仓库，是**不同的工具**：它装 x-cmd 并且**默认你会用 x-cmd 命令做剩下的事**（`x gitb`、`x ws` 等）—— 即"1 action + x-cmd 命令 = 完整 CI"。整个 CI 是 x-cmd 驱动的时候用它。

## 约定

- **仓库命名**：`x-cmd-action/<name>`
- **一仓一个 action**。不搞 monorepo
- **纯 shell only**。没有 TypeScript、没有 JS bundle、没有 `actions/setup-*` 装工具链
- **v1+ tag 稳定版**，`@main` 追踪前沿
- 全 org **Apache 2.0**
- **欢迎 PR** —— 加新 action 或改进现有 action 都行

## 添加新 action

1. `gh repo create x-cmd-action/<name> --public`
2. 写 `action.yml` + （可选）`lib/<script>.sh`
3. 在本 `profile/README.md` 的表格里加一行
4. 提 PR

## 相关链接

- [x-cmd/action](https://github.com/x-cmd/action) —— 全套 CI bootstrap（兄弟仓库，不在这 org）
- [x-cmd](https://github.com/x-cmd/x-cmd) —— 底层 shell 库
- [x-cmd/get](https://github.com/x-cmd/get) —— x-cmd 安装器