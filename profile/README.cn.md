# x-cmd-action

[x-cmd](https://github.com/x-cmd/x-cmd) 生态的 GitHub Actions。这个 org 下每个 action 都是**纯 shell** —— 没有 Node.js runtime、没有 JS bundle、没有嵌套 action 依赖。

[English](./README.md)

## Actions

| Action | 说明 | 需要 x-cmd？ | 最新版 |
| --- | --- | --- | --- |
| [`x-cmd`](./x-cmd) | 安装 x-cmd 到 `~/.x-cmd.root/`。单一职责、幂等、暴露 `outputs.root` 供下游链式用。 | — *(就是 x-cmd)* | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`checkout`](./checkout) | 纯 shell `git checkout`。`actions/checkout` 的直接替代 —— 20 个 input，同名表面。**不需要 x-cmd** —— 只用 `git`、`ssh-agent`、`ssh-keyscan`。 | ❌ 不需要 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`gitmirror`](./gitmirror) | 跨平台 git repo 镜像（GitHub ↔ Gitee ↔ Codeberg）。三种 list 来源、扇出并发。内部调 `x gitb backup`。 | ✅ 需要 | ![v1](https://img.shields.io/badge/v1-stable-green) |

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