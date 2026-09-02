# x-cmd-action

[x-cmd](https://github.com/x-cmd/x-cmd) 生态的 GitHub Actions。这个 org 下每个 action 都是**纯 shell** —— 没有 Node.js runtime、没有 JS bundle、没有嵌套 action 依赖。选 shell 不选 Node.js 是故意的:bash 和 coreutils 不会像 Node.js 那样每半年发一个大版本。今天写的 `v1` action 在 2030 年的 runner 上照样能跑。

本组织还提供 **AI 工具集** —— Layer 3 actions 通过 x-cmd 的 `x ai request` 接口,把 GitHub 原生制品(issues、PRs、comments、diffs)交给 LLM。自动给新 issue 打 label,每次 push 都发 PR review 草稿,生成每周 changelog,从已关闭 bug 提取 post-mortem。AI 不是 demo —— 是大多数用户采用本组织的原因。

[English](./README.md)

## Actions

三层。Layer 1 让 runner 接近本地开发环境;Layer 2 是基于 Layer 1 构建的高阶自动化;**Layer 3 通过 `x ai request` 把 GitHub 原生制品交给 LLM**。

### Layer 1 — Basic Setup

可组合的环境配置 action。各做一件事,按需取用。

| Action | 说明 | 最新版 |
| --- | --- | --- |
| [`x-cmd`](https://github.com/x-cmd-action/x-cmd) | 安装 x-cmd 到 `~/.x-cmd.root/`。单一职责、幂等。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`this-repo`](https://github.com/x-cmd-action/this-repo) | 纯 shell 最精简 checkout:把触发 repo 克隆到 `$GITHUB_WORKSPACE`,用 runner 的 token。6 个 input。`checkout` 太重时用它。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ssh`](https://github.com/x-cmd-action/ssh) | 纯 shell `ssh-agent` setup + `known_hosts` + 可选 key add。从 `x-cmd/action` 抽出。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`docker`](https://github.com/x-cmd-action/docker) | 纯 shell `docker login` + `docker buildx init`。从 `x-cmd/action` 抽出。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`gitconfig`](https://github.com/x-cmd-action/gitconfig) | 纯 shell **全局** git config。默认设 `user.name`/`user.email`;`config` input 给 `~/.gitconfig` 加 `[include]`。位置无关 —— 不提供 `local-config`(repo-scoped overlay 用 `checkout` / `this-repo`)。 | ![v1](https://img.shields.io/badge/v1-stable-green) |

### Layer 2 — Common Functions

更高阶的自动化。每个把 Layer 1 action(和 x-cmd 命令)组合成一个常用 workflow 任务。

| Action | 说明 | 最新版 |
| --- | --- | --- |
| [`checkout`](https://github.com/x-cmd-action/checkout) | 纯 shell `git checkout`。22 个 input,与 `actions/checkout@v4` 1:1 对齐 + 3 个 x-cmd 增强(`known-hosts-url`、`fetch-additional`、`local-config`)。用 `GIT_SSH_COMMAND` + temp 文件 + 硬编码 `github.com` 公钥(学 `actions/checkout`)。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`gitmirror`](https://github.com/x-cmd-action/gitmirror) | 跨平台同步 repo(GitHub ↔ Gitee ↔ Codeberg)。三种 list 来源、扇出并发。需要 x-cmd(用 `x gitb backup`)。 | ![v1](https://img.shields.io/badge/v1-stable-green) |

### Layer 3 — AI 辅助

对 GitHub 原生制品(issues、PRs、comments、diffs)做转换的 AI actions。每个用 `x gh` 读,问 `x ai request`,再用 `x gh` 写回去。无 AI SDK,无 Node 依赖 —— 整个 AI 调用就是一行 `x ai request`。

| Action | 它能帮你做什么 | 最新版 |
| --- | --- | --- |
| [`ai/triage`](https://github.com/x-cmd-action/ai/tree/main/triage) | **自动路由新 issue**。读正文 + 已有 labels,AI 返回 `type / priority / area / labels / summary`,bot 发 summary 并贴建议的 labels。帮维护者省下每个 issue 开头的 5 分钟。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/reply`](https://github.com/x-cmd-action/ai/tree/main/reply) | **一线应答**。监听 issue/comment 正文里的 `@x`,加 reaction 并发回复。结合 FAQ,bot 引用 FAQ 中回答问题的章节。**不需要 AI token。** | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/review`](https://github.com/x-cmd-action/ai/tree/main/review) | **每次 push 都 PR review**。用 `gh pr diff` 拿 diff,AI 返回 Security / Style / Suggestions / Summary,bot 发结构化评论。准到能在人类 reviewer 之前抓明显问题。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/changelog`](https://github.com/x-cmd-action/ai/tree/main/changelog) | **每周社区摘要**。`schedule: cron`。收集过去 N 天关闭的 Issue + 合并的 PR,AI 按 `Features / Fixes / Performance / Docs / Other` 分组,写 `CHANGELOG.md`。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/translate`](https://github.com/x-cmd-action/ai/tree/main/translate) | **按需 i18n**。读 Markdown 文件,翻译到目标语言,写出去。Markdown 友好 —— code block 不动,URL 和专有名词保留。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/spec`](https://github.com/x-cmd-action/ai/tree/main/spec) | **RFC + post-mortem 草稿**。`mode: rfc` 读 feature request,产出结构化 RFC。`mode: postmortem` 从已关闭 bug 提取时间线 + 根因 + 教训。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`ai/commit`](https://github.com/x-cmd-action/ai/tree/main/commit) | **Conventional Commits 强制**。`mode: check` 校验 PR commits,不合规给重写建议。`mode: generate` 从 staged diff 写合规 commit message。跟 `ai/changelog` 是天生搭档。 | ![v1](https://img.shields.io/badge/v1-stable-green) |
| [`mneme`](https://github.com/x-cmd-action/mneme) | **AI 记忆层**。跨 workflow run 持久化 + 检索 LLM context(默认 backend:GitHub Issue)。让 PR #123 的 `ai/review` 记住 issue #42 的 `ai/triage` 说了什么 —— 没有 `mneme`,每个 Layer 3 action 都从零 context 开始。 | ![v1](https://img.shields.io/badge/v1-stable-green) |

Layer 3 的全部内部细节(per-action 设计笔记、roadmap、story 归档)放在私有的 [`mneme`](https://github.com/x-cmd-action/mneme) repo。

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
   x ai request     x mneme store/retrieve
   (call LLM)      (cross-run memory)
        │                   │
        └─────────┬─────────┘
                  ▼
            comment / labels / CHANGELOG.md
```

- **`x-cmd`** —— 装 x-cmd。想要 `x` 可用时用。
- **`this-repo`** —— 最精简 clone 到 `$GITHUB_WORKSPACE`。`checkout` 太重时用它。
- **`checkout`** —— 完整 `actions/checkout` 对齐的 clone(SSH、sparse、filter、fetch-additional、known-hosts-url、gitconfig)。需要这些时用这个。
- **`gitmirror`** —— 跨平台单向复制。
- **`ai/*`** —— 自动 label、一线 reply、PR review、每周 changelog、i18n、RFC 草稿、commit 强制。配合 `mneme` 实现跨 run 记忆。

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
- [mneme](https://github.com/x-cmd-action/mneme) —— 私有组织级记忆 + 设计笔记(Layer 3 内部细节在这里)