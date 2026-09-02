# x-cmd-action

把 [x-cmd shell 库](https://github.com/x-cmd/x-cmd)(以及它包装的 AI 工具集)带进 CI runner 的 GitHub Actions。纯 shell —— 无 Node.js、无 bundled JS、无嵌套 action 依赖。

这些 actions 把 runner 配置好 —— 装 x-cmd、配 git/ssh/docker、clone repo,按需选用 —— 让可重复的运维活能真正跑起来。同时它们辅助维护者管理 GitHub 原生制品(自动给新 issue 打 label,每次 push 发 PR review 草稿,生成每周 changelog,从已关闭 bug 提取 post-mortem)。**AI 出草稿,人类做决策。**

**设计原则**。可重复的运维活在可移植的 shell / Python / JS 脚本里(`x gitb backup`、`x gh`、`x ws` 等) —— 这些脚本你在笔记本、cron、任何 CI 上都能跑。这里的 actions 是 **那些脚本的薄包装**,不是只能在 GitHub Actions 里跑的黑盒逻辑。workflow 离开 GitHub,这些工作跟着走。

[English](./README.md)

## Layer 1 — 把 runner 配成跟笔记本一样

| Action | 说明 | 最新版 |
| --- | --- | --- |
| [`x-cmd`](https://github.com/x-cmd-action/x-cmd) | 装 x-cmd 到 `~/.x-cmd.root/`。单一职责、幂等。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`this-repo`](https://github.com/x-cmd-action/this-repo) | 纯 shell 最精简 checkout:把触发 repo 克隆到 `$GITHUB_WORKSPACE`。6 个 input。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ssh`](https://github.com/x-cmd-action/ssh) | 纯 shell `ssh-agent` 配置 + `known_hosts` + key add。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`docker`](https://github.com/x-cmd-action/docker) | 纯 shell `docker login` + `buildx init`。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`gitconfig`](https://github.com/x-cmd-action/gitconfig) | 纯 shell **全局** git config(name/email + `[include]` 引入 config 文件)。 | ![v1](https://img.shields.io/badge/v1-stable-green) |

## Layer 2 — 常用 CI 任务

| Action | 说明 | 最新版 |
| --- | --- | --- |
| [`checkout`](https://github.com/x-cmd-action/checkout) | 纯 shell `git checkout`。22 个 input,与 `actions/checkout@v4` 1:1 对齐 + 3 个 x-cmd 增强。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`gitmirror`](https://github.com/x-cmd-action/gitmirror) | 跨平台同步 repo(GitHub ↔ Gitee ↔ Codeberg)。 | ![v1](https://img.shields.io/badge/v1-stable-green) |

## Layer 3 — AI 工具集(LLM 走 x-cmd 的 `x ai request`)

每个 Layer 3 action 用 `x gh` 读 GitHub 制品,问 `x ai request`,再用 `x gh` 写回去。无 AI SDK,无 Node 依赖 —— 整个 AI 调用就是一行 `x ai request`。**AI 出草稿,人类定稿**。每个 Layer 3 的输出都是维护者审阅编辑的起点,不是直接推到 `main` 的自治 agent。

| Action | 它能帮你做什么 | 最新版 |
| --- | --- | --- |
| [`ai/triage`](https://github.com/x-cmd-action/ai/tree/main/triage) | **自动给新 issue 打 label**。读正文 + 已有 labels,AI 返回 `type / priority / area / labels / summary`,bot 发 summary 并贴建议的 labels。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/reply`](https://github.com/x-cmd-action/ai/tree/main/reply) | **一线应答**。监听 issue/comment 正文里的 `@x`,加 reaction 并发回复。结合 FAQ,bot 引用 FAQ 中回答问题的章节。**不需要 AI token。** | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/review`](https://github.com/x-cmd-action/ai/tree/main/review) | **每次 push 都 PR review**。用 `gh pr diff` 拿 diff,AI 返回 Security / Style / Suggestions / Summary,bot 发结构化评论。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/changelog`](https://github.com/x-cmd-action/ai/tree/main/changelog) | **每周社区摘要**。`schedule: cron`。收集过去 N 天关闭的 Issue + 合并的 PR,AI 按 `Features / Fixes / Performance / Docs / Other` 分组,写 `CHANGELOG.md`。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/translate`](https://github.com/x-cmd-action/ai/tree/main/translate) | **按需 i18n**。读 Markdown 文件,翻译到目标语言。Markdown 友好 —— code block 不动。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/spec`](https://github.com/x-cmd-action/ai/tree/main/spec) | **RFC + post-mortem 草稿**。`mode: rfc` 或 `mode: postmortem`。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/commit`](https://github.com/x-cmd-action/ai/tree/main/commit) | **Conventional Commits 强制**。`mode: check` 校验 PR commits;`mode: generate` 写合规 commit message。跟 `ai/changelog` 是搭档。 | ![v1](https://img.shields.io/badge/v1-stable-green) |

跨 run 记忆(让 `ai/review` 记住链接 issue 上 `ai/triage` 说了什么)在内部处理 —— Layer 3 用户不用自己接,直接拿有用草稿。

## 快速案例

把 runner 配成跟笔记本一样:

```yaml
steps:
  - uses: x-cmd-action/x-cmd@v1
  - uses: x-cmd-action/this-repo@v1
  - uses: x-cmd-action/gitconfig@v1
    with:
      email: bot@example.com
```

自动给新 issue 打 label(AI token 还没配的话先在 repo secrets 里加 `MINIMAX_TOKEN`):

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
        with: { model: minimax, apply-labels: 'true' }
        env: { MINIMAX_TOKEN: ${{ secrets.MINIMAX_TOKEN }} }
```

监听 `@x` 触发,引用 FAQ 回复用户(不需要 AI token):

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
```

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
   │          checkout     ← Layer 2:完整 actions/checkout 对齐
   │          (clone+more)
   ▼
 ssh, docker, gitconfig
 (各加一项)
```

Layer 3 (AI) 是这样:

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
   x ai request     跨 run 记忆
   (call LLM)      (内部处理)
        │                   │
        └─────────┬─────────┘
                  ▼
            comment / labels / CHANGELOG.md
```

- **`x-cmd`** —— 装 x-cmd。想要 `x` 可用时用。
- **`this-repo`** —— 最精简 clone 到 `$GITHUB_WORKSPACE`。`checkout` 太重时用它。
- **`checkout`** —— 完整 `actions/checkout` 对齐的 clone(SSH、sparse、filter、fetch-additional、known-hosts-url、gitconfig)。需要这些时用这个。
- **`gitmirror`** —— 跨平台单向复制。
- **`ai/*`** —— 自动 label、一线 reply、PR review、每周 changelog、i18n、RFC 草稿、commit 强制。跨 run 记忆内部处理,用户不用接。

这些是同级 —— 需要哪个就拿哪个,可以自由组合。

[`x-cmd/action`](https://github.com/x-cmd/action) 在独立仓库,是**不同的工具**:它装 x-cmd 并且**默认你会用 x-cmd 命令做剩下的事**(`x gitb`、`x ws` 等) —— 即"1 action + x-cmd 命令 = 完整 CI"。整个 CI 是 x-cmd 驱动的时候用它。

## 约定

- **仓库命名**:本组织 actions 是 `x-cmd-action/<name>`。Layer 3 子命令放在 `x-cmd-action/ai` 子目录下,通过 `x-cmd-action/ai/<subcmd>` 引用。
- **一仓一个 action**。Layer 3 是唯一例外:`x-cmd-action/ai` 是 monorepo 含七个子命令,因为它们共用 `x ai request` 和 `x gh` glue。
- **纯 shell only**。没有 TypeScript、没有 JS bundle、没有 `actions/setup-*` 装工具链
- **v1+ tag 稳定版**;`@main` 追踪前沿
- 全 org **Apache 2.0**
- **欢迎 PR** —— 加新 action 或改进现有 action 都行

## 添加新 action

1. `gh repo create x-cmd-action/<name> --public`
2. 写 `action.yml` + (可选)`lib/<script>.sh`
3. 在本 `profile/README.md` 的表格里加一行
4. 提 PR

加 Layer 3 子命令时,改加到 `x-cmd-action/ai/<subname>/` 并在 Layer 3 表格里 link。

## 相关链接

- [x-cmd/action](https://github.com/x-cmd/action) —— 全套 CI bootstrap(兄弟仓库,不在这 org)
- [x-cmd](https://github.com/x-cmd/x-cmd) —— 底层 shell 库
- [x-cmd/get](https://github.com/x-cmd/get) —— x-cmd 安装器
- (无对外公开的跨 run 记忆 action —— 内部处理)