# x-cmd-action

[x-cmd](https://github.com/x-cmd/x-cmd) 生态的 GitHub Actions。这个 org 下每个 action 都是**纯 shell** —— 没有 Node.js runtime、没有 JS bundle、没有嵌套 action 依赖。

[English](./README.md)

## Actions

两层。Layer 1 让 runner 接近本地开发环境；Layer 2 是基于 Layer 1 构建的高阶自动化。

### Layer 1 — Basic Setup

可组合的环境配置 action。各做一件事，按需取用。

| Action | 说明 | 最新版 |
| --- | --- | --- |
| [`x-cmd`](./x-cmd) | 安装 x-cmd 到 `~/.x-cmd.root/`。单一职责、幂等。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`checkout`](./checkout) | 纯 shell `git checkout`。20 个 input，同 `actions/checkout` 表面。**不依赖 x-cmd** —— 只用 `git`、`ssh-agent`、`ssh-keyscan`。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ssh`](./ssh) | 纯 shell `ssh-agent` setup + `known_hosts` + 可选 key add。从 `x-cmd/action` 抽出。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`docker`](./docker) | 纯 shell `docker login` + `docker buildx init`。从 `x-cmd/action` 抽出。 | ![v1](https://img.shields.io/badge/v1-stable-green) |

### Layer 2 — Common Functions

自成体系的自动化。每个解决一个重复出现的 workflow 任务，底层是 Layer 1 + x-cmd 模块。

| Action | 说明 | 最新版 |
| --- | --- | --- |
| [`gitmirror`](./gitmirror) | 跨平台同步 repo（GitHub ↔ Gitee ↔ Codeberg）。三种 list 来源、扇出并发。需要 x-cmd（用 `x gitb backup`）。 | ![v1](https://img.shields.io/badge/v1-stable-green) |

更多计划中的 action（`ghwatch`、`ghissuereply`、`ghissuegold`、`webmonitor`、`hnmonitor`）—— 见[内部路线图](https://github.com/x-cmd-action/.github/blob/main/README.md)。

## 它们怎么配合

```
┌─────────────────────────────────────────────────────┐
│  你的 workflow                                       │
└─────────────────┬───────────────────────────────────┘
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
   x-cmd      checkout    gitmirror
   (装)       (clone)     (同步)
```

- **`x-cmd`** —— 装 x-cmd。想要 `x` 可用时用。
- **`checkout`** —— 把 repo 克隆进 workspace。不想引 Node.js 时替代 `actions/checkout`。
- **`gitmirror`** —— 跨平台单向复制。在 Gitee/Codeberg 维护镜像时用。

这三个是同级 —— 需要哪个就拿哪个，可以自由组合。

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