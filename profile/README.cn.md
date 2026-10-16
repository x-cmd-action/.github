# x-cmd-action

把 [x-cmd shell 库](https://github.com/x-cmd/x-cmd) 带进 GitHub Actions 的一套 **纯 shell** action 合集:帮你装好 runner 环境(装 x-cmd、clone 仓库、配好 git/ssh/docker),也能用 AI 自动维护 Issues / PR。无 Node.js 依赖,每个 action 一行 `uses:` 即可接入。

[English](./README.md)

## 这里有什么

### 环境准备 —— 装 x-cmd、clone 仓库、配好 git / ssh / docker

| Action | 说明 | 最新版 |
| --- | --- | --- |
| [`x-cmd`](https://github.com/x-cmd-action/x-cmd) | 装 x-cmd 到 `~/.x-cmd.root/`,之后就能用 `x ai`、`x gh` 等命令。单一职责、幂等。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`this-repo`](https://github.com/x-cmd-action/this-repo) | 最精简 checkout:把触发 repo 克隆到 `$GITHUB_WORKSPACE`。6 个 input。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ssh`](https://github.com/x-cmd-action/ssh) | 纯 shell `ssh-agent` 配置 + `known_hosts` + key add。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`docker`](https://github.com/x-cmd-action/docker) | 纯 shell `docker login` + `buildx init`。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`gitconfig`](https://github.com/x-cmd-action/gitconfig) | 纯 shell **全局** git config(name/email + `[include]` 引入 config 文件)。 | ![v1](https://img.shields.io/badge/v1-stable-green) |

### 常用 CI 任务

| Action | 说明 | 最新版 |
| --- | --- | --- |
| [`checkout`](https://github.com/x-cmd-action/checkout) | 完整版 `git checkout`:22 个 input,与 `actions/checkout@v4` 1:1 对齐 + 3 个 x-cmd 增强(SSH、sparse、filter 等)。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`gitmirror`](https://github.com/x-cmd-action/gitmirror) | 跨平台同步 repo(GitHub ↔ Gitee ↔ Codeberg)。 | ![v1](https://img.shields.io/badge/v1-stable-green) |

### AI 工具集 —— 自动维护 Issues / PR

新 issue 自动打标签、PR 自动 review、每周自动生成 changelog。token 从环境变量读取(`MINIMAX_API_KEY`、`DEEPSEEK_API_KEY`、`OPENAI_API_KEY` 等),在 repo secrets 里配一个即可。**AI 出草稿,人类做决策。**

| Action | 它能帮你做什么 | 最新版 |
| --- | --- | --- |
| [`ai/triage`](https://github.com/x-cmd-action/ai/tree/main/triage) | **自动给新 issue 打 label**。读正文 + 已有 labels,AI 返回 `type / priority / area / labels / summary`,bot 发 summary 并贴建议的 labels。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/reply`](https://github.com/x-cmd-action/ai/tree/main/reply) | **一线应答**。监听 issue/comment 正文里的 `@x`,加 reaction 并发回复。默认发固定回复;设 `use_ai: 'true'` 后由 AI 根据上下文生成回复。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/review`](https://github.com/x-cmd-action/ai/tree/main/review) | **PR 自动 review**。用 `gh pr diff` 拿 diff,AI 返回 Security / Style / Suggestions / Summary,bot 发结构化评论。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/changelog`](https://github.com/x-cmd-action/ai/tree/main/changelog) | **每周社区摘要**。`schedule: cron`。收集过去 N 天关闭的 Issue + 合并的 PR,AI 按 `Features / Fixes / Performance / Docs / Other` 分组,写 `CHANGELOG.md`。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/translate`](https://github.com/x-cmd-action/ai/tree/main/translate) | **按需 i18n**。读 Markdown 文件,翻译到目标语言。Markdown 友好 —— code block 不动。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/spec`](https://github.com/x-cmd-action/ai/tree/main/spec) | **RFC + post-mortem 草稿**。`mode: rfc` 或 `mode: postmortem`。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/commit`](https://github.com/x-cmd-action/ai/tree/main/commit) | **Conventional Commits 强制**。`mode: check` 校验 PR commits;`mode: generate` 写合规 commit message。跟 `ai/changelog` 是搭档。 | ![v1](https://img.shields.io/badge/v1-stable-green) |

跨 run 记忆(让 `ai/review` 记住链接 issue 上 `ai/triage` 说了什么)在内部处理 —— 用户不用自己接,直接拿有用草稿。

## 快速案例

装好基础环境(x-cmd、仓库、git 身份):

```yaml
steps:
  - uses: x-cmd-action/x-cmd@v1
  - uses: x-cmd-action/this-repo@v1
  - uses: x-cmd-action/gitconfig@v1
    with:
      email: bot@example.com
```

自动给新 issue 打 label(先在 repo secrets 里加 `MINIMAX_API_KEY`,也支持 `DEEPSEEK_API_KEY` / `OPENAI_API_KEY` 等):

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

监听 `@x`,AI 生成回复(不需要 AI 的话去掉 `use_ai` 和 `env`,发固定文案):

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
          use_ai: 'true'           # 由 AI 根据上下文生成回复
        env:
          MINIMAX_API_KEY: ${{ secrets.MINIMAX_API_KEY }}
```

## 设计原则

1. **纯 shell** — 不用 Node.js、不带 JS bundle、不嵌 action 依赖。绕开 Node / Python / 库的版本升级导致的 action 频繁发版。
2. **可重复的运维活在 shell / Node.js / Python 里。** 这些 action 只负责把 runner 配好 —— 装 x-cmd、配 git/ssh/docker、clone repo。真正干活的代码在 `x gitb backup`、`x gh`、`x ws` 这些脚本里 —— 你在笔记本、cron、任何 CI 都能跑。workflow 哪天离开 GitHub,这些活也跟着走。
3. **轻量、安全的 AI 辅助维护。** 自动给新 issue 打 label,每次 push 发 PR review 草稿,生成每周 changelog,从已关闭 bug 提取 post-mortem。**AI 出草稿,人类做决策。** 每个 AI 输出都是维护者审阅、编辑的起点,不是直接推到 `main` 的自治 agent。

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
   │          checkout     ← 完整 actions/checkout 对齐
   │          (clone+more)
   ▼
 ssh, docker, gitconfig
 (各加一项)
```

AI 工具集是这样:

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

- **仓库命名**:本组织 actions 是 `x-cmd-action/<name>`。AI 子命令放在 `x-cmd-action/ai` 子目录下,通过 `x-cmd-action/ai/<subcmd>` 引用。
- **一仓一个 action**。AI 工具集是唯一例外:`x-cmd-action/ai` 是 monorepo 含七个子命令,因为它们共用 `x ai` 和 `x gh` glue。
- **纯 shell only**。没有 TypeScript、没有 JS bundle、没有 `actions/setup-*` 装工具链
- **v1+ tag 稳定版**;`@main` 追踪前沿
- 全 org **Apache 2.0**
- **欢迎 PR** —— 加新 action 或改进现有 action 都行

## 添加新 action

1. `gh repo create x-cmd-action/<name> --public`
2. 写 `action.yml` + (可选)`lib/<script>.sh`
3. 在本 `profile/README.md` 的表格里加一行
4. 提 PR

加 AI 子命令时,改加到 `x-cmd-action/ai/<subname>/` 并在 AI 工具集表格里 link。

## FAQ

### AI token 怎么配?支持哪些 provider?

所有调用 AI 的子命令都从环境变量读 token,在 workflow 里用 `env:` + secrets 传入即可,如 `MINIMAX_API_KEY: ${{ secrets.MINIMAX_API_KEY }}`。x-cmd 按 key 自动识别 provider —— 支持 `MINIMAX_API_KEY`、`DEEPSEEK_API_KEY`、`GEMINI_API_KEY`、`XAI_API_KEY`、`MISTRAL_API_KEY`、`MOONSHOT_API_KEY`、`OPENAI_API_KEY` 等。例外:`ai/reply` 用固定文案回复时、`ai/commit` 的 `check` 模式只做格式校验时,不需要 token。

### AI 会直接改我的仓库、自动 push 到 main 吗?

不会。所有 AI 输出都是**草稿**:issue/PR 上的评论、建议的 labels、写到工作区的文件。没有任何 action 会自主提交或推送 —— 合不合、改不改,由人决定。

### 只能用在 GitHub Actions 里吗?

Action 是 GitHub Actions 的封装,但核心逻辑是 x-cmd 命令:本地装一个 x-cmd,`x ai triage`、`x ai review` 等都能直接跑。可以在本地调通 prompt 和效果,再上 CI。

### ai/triage 建议的 label 在仓库里不存在怎么办?

会自动创建缺失的 label 再贴上,不需要事先手工建标签。

### 为什么要鼓励用 shell / Node.js / Python 写流程,而不是用 `action.yml` 组合?

可重复的运维活在可移植的 shell / Python / JS 脚本里(`x gitb backup`、`x gh`、`x ws` 等) —— 这些脚本你在笔记本、cron、任何 CI 上都能跑。这里的 actions 是 **那些脚本的薄包装**,不是只能在 GitHub Actions 里跑的黑盒逻辑。workflow 离开 GitHub,这些工作跟着走。

## 相关链接

- [x-cmd/action](https://github.com/x-cmd/action) —— 全套 CI bootstrap(兄弟仓库,不在这 org)
- [x-cmd](https://github.com/x-cmd/x-cmd) —— 底层 shell 库
- [x-cmd/get](https://github.com/x-cmd/get) —— x-cmd 安装器
- (无对外公开的跨 run 记忆 action —— 内部处理)
