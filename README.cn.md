# x-cmd-action (内部)

> **仅供组织维护者参考。** 本文件是内部视图 — 分类 actions、追踪规划中的工作、记录设计意图。对外的公开版本是 [`profile/README.md`](./profile/README.md)。

[English version](./README.md)

## 受众与目的

本组织构建面向 **超级个体** 的 GitHub Actions — 那些在 GitHub Actions 上跑自己个人工作流流水线的人。这里的 actions 设计成可组合的,把一个全新的 CI runner 拉近到本地开发环境的体验,让用户能低成本地迭代个人自动化 — 而且,**越来越重要的是,让 AI 在用户仓库里充当一线维护者**。

我们把 actions 分为三层。Layer 1 和 Layer 2 负责把 runner 配对(SSH、git config、checkout、监控)。**Layer 3 负责把 GitHub 原生制品(issues、PRs、comments、diffs)通过 x-cmd 的 `x ai request` 接口交给 LLM** — 自动给新 issue 打 label、从 FAQ 里找答案回复用户、每次 push 都发 PR review 草稿、从已关闭 bug 提取 post-mortem。AI 不是 demo 功能,是大多数用户采用本组织的原因。

## Layer 1 — 基础设置

把 runner 拉近本地开发环境的 actions。**可组合、互相独立。** 按需选用;目标是"这个 runner 用起来像我的笔记本"。

| Action | 状态 | 用途 |
| --- | --- | --- |
| [`x-cmd`](https://github.com/x-cmd-action/x-cmd) | ✅ shipped (v1) | 把 x-cmd 装到 `~/.x-cmd.root/` |
| [`this-repo`](https://github.com/x-cmd-action/this-repo) | ✅ shipped (v1) | 最小化 clone 触发 repo 到 `$GITHUB_WORKSPACE`。只支持 token,无 SSH、无额外配置,6 个 inputs。 |
| [`ssh`](https://github.com/x-cmd-action/ssh) | ✅ shipped (v1) | ssh-agent + known_hosts + key add |
| [`docker`](https://github.com/x-cmd-action/docker) | ✅ shipped (v1) | docker login + buildx init |
| [`gitconfig`](https://github.com/x-cmd-action/gitconfig) | ✅ shipped (v1) | 全局 git config(name/email + `[include]` 引入 config 文件)。位置无关 — 没有 repo-scoped 逻辑。 |

**为什么有这一层。** x-cmd 的 `gitb backup` 需要 ssh-keyscan。`x gh` 需要 GitHub token。大多数基于 x-cmd 的 actions 都依赖这些配置。把它们拆出来,让用户按需组合,不为不需要的功能付钱。

**为什么 `gitconfig` 没有 `local-config` input。** `local-config` 写入特定 repo 的 `.git/config`,隐式依赖 cwd(某个特定 repo)。这种耦合属于"知道自己在 checkout 哪个 repo"的 action — `checkout` / `this-repo`。全局 config action 应该是位置无关的。详见 `gitconfig/README.md` 的 FAQ。

## Layer 2 — 常用功能

基于 Layer 1 构建的自包含自动化。每个解决一个反复出现的工作流问题。

| Action | 状态 | 用途 |
| --- | --- | --- |
| [`checkout`](https://github.com/x-cmd-action/checkout) | ✅ shipped (v1) | pure-shell `git checkout`。22 个 inputs,与 `actions/checkout@v4` 1:1 兼容 + 3 个 x-cmd 增强(`known-hosts-url`、`fetch-additional`、`local-config`)。用 `GIT_SSH_COMMAND` + 临时文件(模仿 actions/checkout)。 |
| [`gitmirror`](https://github.com/x-cmd-action/gitmirror) | ✅ shipped (v1) | 跨 GitHub ↔ Gitee ↔ Codeberg 同步你关注的 repos |
| `ghwatch` | 🚧 TODO | 关注项目上的 issues 与 releases |
| `ghissuereply` | 🚧 TODO | 给新 issue 起草快速回复 |
| `ghissuegold` | 🚧 TODO | 从 issue 线程里挖有用模式/答案 |
| `webmonitor` | 🚧 TODO | 通用 URL/diff 监控 |
| `hnmonitor` | 🚧 TODO | HN 头条监控 |

**为什么 `checkout` 放在这一层。** 它把 token/SSH/sparse/filter/identity 组合在一个 action — 这正是 Layer 2 的定位(基于 Layer 1 的反复性 workflow 任务)。Layer 1 的 `this-repo` 留给那些只想要"给我这个 repo,不要花哨"的用户。

**为什么有这一层。** 常用功能通常都可以表达为 "Layer 1 setup + 一个 x-cmd 脚本"。我们打包好,让用户不必每次重新发现正确的 x-cmd 咒语。内部每个都是 **x-cmd 命令的薄 shell 包装** — 绝不重新实现。

## Layer 3 — AI 辅助(基于 Layer 1 + 2)

本组织采用 x-cmd 不只是因为 shell 原语。x-cmd 还把整个 **AI provider 生态** 包装在单一稳定接口后面 — `x ai request --model <provider>` — 让每个 AI action 享有跟 Layer 1/2 一样的、无依赖、pure-shell 的接入路径。

Layer 3 actions 是 **对 GitHub 原生制品(issues、PRs、comments、diffs)做转换的 AI 包装**。它们很薄:每个用 `x gh` 读制品,问 `x ai` 转换,再用 `x gh` 写回去。同样的 delegation 原则 — 绝不重新实现 `x ai request` 已经做的事。

| Action | 状态 | 用途 |
| --- | --- | --- |
| Action | 状态 | 用途 |
| --- | --- | --- |
| [`ai/triage`](https://github.com/x-cmd-action/ai/triage@v1) | ✅ shipped (v1) | **自动路由新 issue**。读新 issue 的正文 + 已有 labels,问 AI 要 `type / priority / area / labels / summary`,把 summary 发评论,把建议的 labels 贴上去。帮维护者省下每个 issue 开头的 5 分钟。 |
| [`ai/reply`](https://github.com/x-cmd-action/ai/reply@v1) | ✅ shipped (v1) | **一线应答**。监听 issue/comment 正文里的 `@x`(严格词边界,可配置),加 reaction 并发固定回复。结合 `README.md` / `docs/` 里的 FAQ,bot 可以引用 FAQ 中回答问题的章节。**不需要 AI token。** |
| [`ai/review`](https://github.com/x-cmd-action/ai/review@v1) | ✅ shipped (v1) | **每次 push 都 PR review**。用 `gh pr diff` 拿 diff,问 AI 要 Security / Style / Suggestions / Summary,发结构化评论。便宜到每个 PR 都能跑;也准到能在人类 reviewer 之前抓明显问题。 |
| [`ai/changelog`](https://github.com/x-cmd-action/ai/changelog@v1) | ✅ shipped (v1) | **每周社区摘要**。在 `schedule: cron` 触发。收集过去 N 天关闭的 Issue + 合并的 PR,让 AI 按 `Features / Fixes / Performance / Docs / Other` 分组,写 `CHANGELOG.md`。零成本的每周更新。 |
| [`ai/translate`](https://github.com/x-cmd-action/ai/translate@v1) | ✅ shipped (v1) | **按需 i18n**。读 Markdown 文件,翻译到目标语言,写出去。Markdown 友好 — code block 不动,URL 和专有名词保留。适合"我不懂中文,但想要 README 的中文版"。 |
| [`ai/spec`](https://github.com/x-cmd-action/ai/spec@v1) | ✅ shipped (v1) | **RFC + post-mortem 草稿**。`mode: rfc` 读 feature request issue,产出结构化 RFC(Summary / Motivation / Detailed Design / Alternatives / Drawbacks / Open Questions)。`mode: postmortem` 从已关闭 bug 提取时间线 + 根因 + 教训。维护者拿到草稿;他们改,不是从零写。 |
| [`ai/commit`](https://github.com/x-cmd-action/ai/commit@v1) | ✅ shipped (v1) | **Conventional Commits 强制**。`mode: check` 校验 PR 里每条 commit 是否符合规范,不合规时给重写建议。`mode: generate` 从 staged diff 写一条合规 commit message。跟 `ai/changelog` 是天生搭档 — 历史越干净,自动 changelog 越好。 |
| [`mneme`](https://github.com/x-cmd-action/mneme) | ✅ shipped (v1) | **AI 记忆层**。跨 workflow run 持久化 + 检索 LLM context。默认 backend:GitHub Issue(公开 repo 零成本)。让 PR #123 的 `ai/review` 记住 issue #42(这个 PR 关闭的那个)的 `ai/triage` 说了什么 — 没有 `mneme`,每个 Layer 3 action 都从零 context 开始。 |

### 具体场景(Layer 3 究竟能干什么)

Layer 3 actions 不是"AI 演示" — 每个解决一个具体维护者痛点,这些痛点现在要么手动做要么完全不做。

**1. 自动给新 issue 打 label** — 新 issue 打开时,`ai/triage` 读正文 + 已有 labels,问 AI 要 `type / priority / area / labels / summary`,把 summary 发评论,把建议的 labels 贴上去。维护者不用在每个 issue 上花头 5 分钟做路由。

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

**2. 读 FAQ 回答用户问题** — `ai/reply` 在 issue/comment 正文里监听 `@x`。触发时,加 reaction 并发回复。结合 `README.md` / `docs/faq.md` 里的 FAQ,这成了廉价的"一线应答":bot 引用 FAQ 中回答问题的章节,用户几秒钟就拿到有用回复,不用等维护者。

**3. 自动调研新 issue** — `ai/reply` + 自定义 prompt,可以让 AI 读 issue + 关联的 repo 文件 + 相关历史 issue,然后发"诊断草稿"("这看起来跟 PR #55 修的根因一样,在 Y 文件第 X 行")。维护者审草稿,而不是从零开始。

**4. 每次 push 都 PR code review** — `ai/review` 在 `pull_request: opened / synchronize` 触发,用 `gh pr diff` 拿 diff,发结构化 review(Security / Style / Suggestions / Summary)。便宜到可以每个 PR 都跑;也准到能在人类 reviewer 之前抓明显问题。

```yaml
on:
  pull_request:
    types: [opened, synchronize]
jobs:
  review:
    runs-on: ubuntu-latest
    permissions: { contents: read, pull-requests: write }
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: x-cmd-action/ai/review@v1
        env: { MINIMAX_TOKEN: ${{ secrets.MINIMAX_TOKEN }} }
```

**5. Conventional Commits 强制** — `ai/commit` 在 `mode: check` 下校验 PR 里每条 commit 是否符合 [Conventional Commits](https://www.conventionalcommits.org/)。发现不合规时,AI 给重写建议并发到 PR 评论。跟 `ai/changelog` 是天生搭档 — commit 历史越干净,自动 changelog 越好。

**6. 周社区摘要** — `ai/changelog` 在 `schedule: cron` 触发,收集过去 N 天关闭的 Issue + 合并的 PR,让 AI 按 `Features / Fixes / Performance / Docs / Other` 分组,把结果写到 `CHANGELOG.md`(或 stdout 给其他地方用)。零成本的每周更新。

**7. RFC + post-mortem 提取** — `ai/spec` 在 `mode: rfc` 下读 feature request issue,产出结构化 RFC(Summary / Motivation / Detailed Design / Alternatives / Drawbacks / Open Questions)。在 `mode: postmortem` 下从已关闭 bug 提取时间线 + 根因 + 教训。维护者拿到草稿;他们改,而不是从零写。

**8. 按需 i18n** — `ai/translate` 读 `README.md`,翻译到 `target: zh`,写 `README.zh.md`。Markdown 友好 — code block 不动,URL 与专有名词保留。适合"我不懂中文,但想要 README 的中文版"。

### 为什么这是一层,不只是"AI 功能"

- **稳定接口**。所有 Layer 3 actions 共用同一个 `MINIMAX_TOKEN` env 约定、同一个 composite-action 形态、同一个 `x-cmd-action/x-cmd + x-cmd-action/this-repo` 前缀。学一种模式,拿到七个 actions。
- **Provider-agnostic**。`with: model: minimax | openai:gpt-4 | anthropic:claude-fable-5` — 换 provider 只改 input,不重写。x-cmd 的 `x ai request` 处理路由。
- **跨 run 记忆**。`mneme` 让 PR #123 的 `ai/review` 记住 issue #42(这个 PR 关闭的那个)的 `ai/triage` 说了什么。否则每次 AI run 都从零 context 开始。
- **本地优先**。每个 Layer 3 action 对应一个本地 `x ai <subcmd>` — 笔记本上试一下,再 ship 成 workflow。同一条代码路径,同样的输出。

## 约定

- **纯 shell**。不用 TypeScript / Node.js bundles。
- **一仓一 action**。不用 monorepo。
- **shipped 即 v1 tag** + `@main` 用于 bleeding edge。
- **Apache 2.0** 跨整个组织。
- **`profile/README.md`** 是对外门面;本文件是维护者视角。
- **Scope-appropriate 命名**。本组织的独立 action 用 **unprefixed** input 名称(`username`、`password`、`ssh-key`、`buildx-init`)。action 自己的 scope 就是消歧器。前缀(`docker_username`、`ssh_key`)只出现在 `x-cmd/action`,因为它有 17 个跨多领域的 inputs,需要前缀消歧。拿不准时:把 input 命名成"它在自己的 scope 里是唯一的那一个"。

## FAQ

### 为什么是纯 shell,不是 Node.js?

Node.js 系的 GitHub Actions 生态有一个容易被低估的维护模式:actions "过期" 不是因为它们逻辑变了,而是因为 **Node.js 本身是个移动靶**。Node 16 → 18 → 20 → 22 → 24 每次都 break 一些。`actions/checkout`、`actions/setup-node` 等经常发版,基本就是"bump Node 版本",改 deprecation warning,修 ESM/CJS 互操作。用户看到频繁发版就以为真有变化;其实变化通常是"这个 action 现在跑 Node 24 不是 Node 20"。

纯 shell 完全绕开这点。`bash` 就是 bash。POSIX 工具(`git`、`ssh`、`curl`、`grep`、`sed`、`awk`)不会每六个月发一次大版本。2020 年写的 shell action 在 2030 年的 runner 上还能跑,因为解释器跟工具的稳定时间尺度是几十年。

实际结果:

- **没有版本 churn**。`v1` 就是 `v1`。我们不 bump Node runtime。
- **Tarball 更小**。~3 KB shell vs 几 MB bundled JS。checkout 更快,失败也更快。
- **可审计**。读脚本就看清跑什么。没有转译产物、没有 `node_modules`、不用学 `@actions/*` 命名空间。
- **可 fork / inline**。想改?把 lib shell 文件拷进你自己的 action 改。

Trade-off:用不了 Node 库(`@actions/core`、`@actions/github`、`octokit` 等)。对我们 scope 来说 — `git`、`ssh`、`docker`、`curl`、`jq` 来自 coreutils — 没问题。我们需要 LLM SDK 或复杂 HTTP client 时,**没有**拉 Node 依赖 — 我们 delegate 给 x-cmd(`x http`、`x curl`、`x ai request`)。Layer 3 actions 就是证据:它们是纯 shell,把整个 AI 调用 delegate 给一行 `x ai request`。

组织原则: **ship shell;lean on x-cmd 干任何本来要库的事 — 包括 LLM SDK。**

## 设计原则 — 绝不重写 x-cmd 已有的事

x-cmd 提供了几百个原语(`x gitb backup`、`x gh`、`x repo`、`x eget`、`x env use`、`x curl`、`x sysinfo` 等),打磨了多年。本组织的 Layer 2 actions **delegate 给 x-cmd**,不重写。

### 规则

> 一个 action 是 **x-cmd 命令的薄包装**。它不是 x-cmd 逻辑的 shell 重写。

### 具体来说

#### ❌ 不要用 shell 写 — 让 x-cmd 做

| 你需要... | 不要写... | 用 x-cmd 替代 |
| --- | --- | --- |
| 镜像一个 repo | `git clone --bare` + `git push` | `x gitb backup` |
| 调 GitHub API | 自定义 REST client | `x gh` |
| 取一个 URL | 自定义 HTTP 包装 | `x curl` / `x http` / `x fetch` |
| 解析 JSON | 自定义解析器 | `x jq` / `x json` |
| 装语言工具链 | `apt install` / brew 脚本 | `x env use <lang>` |
| 装二进制 | curl + chmod + PATH hack | `x eget use` |

#### ✅ 可以写 shell — 但只写 action glue

- 把 action inputs 解析成 x-cmd 能理解的结构
- 接好 secrets / 凭据(这是 Layer 1 actions 的活)
- 用解析过的 inputs 调 x-cmd
- 把 x-cmd 输出格式化成 GitHub 风格的 outputs(markdown、`$GITHUB_OUTPUT`、`$GITHUB_STEP_SUMMARY`)
- 实现 GitHub Actions 特有的 glue(matrix、conditionals、env files)

### 例外(什么时候 shell 重写 OK)

**只有一种:GitHub Actions 原生 glue。** x-cmd 跑在 GitHub Actions runtime 外面,所以写不了 `$GITHUB_OUTPUT`、渲染不了 `$GITHUB_STEP_SUMMARY`、算不了 `if:` 表达式、声明不了 matrix strategies、读不了 secrets。这种 orchestration 代码是 action 的活 — 用 shell 写。

**其他的,call x-cmd。** 用这个启发式:*如果一个过程可以原子化并清晰描述,它就应该能表达成一个或多个 x-cmd 命令*。所以:

- "对 LLM X 发 HTTPS 请求" 是原子化的 → `x curl` 能做。不是例外。
- "跨 step 持久化 key/value state" 是原子化的 → x-cmd 有 file/env 原语。不是例外。
- "写 `$GITHUB_OUTPUT`" **不能** 原子化成 x-cmd — 这是 GitHub Actions runtime 的事。例外。

拿不准时:先写 x-cmd 版本。如果 x-cmd 真的做不了,例外适用。

### 为什么这事重要

| 维度 | 重写 | delegate 给 x-cmd |
| --- | --- | --- |
| 代码量 | 每个原语 5+ 行 | 每个调用 1 行 |
| 修 bug | 你自己维护 | x-cmd 维护者修,所有 actions 受益 |
| 性能 | 你写啥是啥 | x-cmd 调优到啥是啥 |
| 行为一致性 | 跟 `x gitb backup` 等分叉 | 100% 跟本地 `x ...` 一致 |
| 测试 | 你写 | x-cmd 的测试覆盖 |

### 反面例子(gitmirror v0)

```bash
# 错 — 重写了 x gitb backup 已经做的事
git clone --bare "$src" "$work/repo.git"
cd "$work/repo.git"
git remote add target "$dest"
git push target --all --tags --force
```

### 正确姿势(gitmirror v1)

```bash
# 对 — 一行,delegate
x gitb backup --force "$src" "$dest"
```

## TODO

### 从 `x-cmd/action` 抽出来

- [x] **`x-cmd-action/ssh`** — 把 ssh-agent / known_hosts setup 从 `x-cmd/action` 抽成独立 action(input: `ssh_key`)。让用户在任意 workflow 里加 SSH 而不用拉整个 `x-cmd/action`。 — **完成,v1 已发。**
- [x] **`x-cmd-action/docker`** — 把 `docker login` + `docker buildx create --use` 从 `x-cmd/action` 抽成独立 action(inputs: `docker_username`、`docker_password`、`docker_buildx_init`)。 — **完成,v1 已发。**

### Layer 2 新 actions

- [ ] **`x-cmd-action/ghwatch`** — `on: schedule: cron: '0 */6 * * *'`。对 input 里的 `repo` 列表,用 `x gh` 拿最近的 issues + releases。JSON 输出到 `$GITHUB_STEP_SUMMARY` 或推到一个 gist。
- [ ] **`x-cmd-action/ghissuereply`** — 给定 issue number, 用可配置模板 + LLM 调用起草回复。用 `x gh` 拉数据,输出可 PR 的 markdown 文件。
- [ ] **`x-cmd-action/ghissuegold`** — 给定一个已关闭 issue(或线程),把 accepted answer / fix 总结成可复用 snippet。PR 到知识库文件。
- [ ] **`x-cmd-action/webmonitor`** — 通用:拿一组 URL,fetch,跟缓存的 snapshot 做 diff,有变化就开 issue。
- [ ] **`x-cmd-action/hnmonitor`** — HN 头条监控。用 `x web` 或 `x curl` 调 HN API,新故事输出成摘要。

### Layer 3 (AI) — 完成

- [x] **`x-cmd-action/ai`** — monorepo,七个子命令(`triage`、`reply`、`review`、`changelog`、`translate`、`spec`、`commit`),每个与本地 `x ai <subcmd>` 一一对应。 — **完成,v1 已发。**
- [x] **`x-cmd-action/mneme`** — AI 记忆层(跨 workflow run 的 store / retrieve / search;默认 backend:GitHub Issue)。 — **完成,v1 已发。**

### Layer 3 未来想法

- [ ] **`ai/rank`** — 按"用户痛度"估计(bug 频率 × 严重度 × 用户影响)给新 issue 排序,让维护者按紧急度而不是到达顺序 triage。
- [ ] **`ai/reply-context`** — 从 `docs/` / `README.md` / 历史已关闭 issue 拉相关 snippet,然后 bot 回答时引用。给文档丰富的 repo 改善 plain `ai/reply` 的体验。
- [ ] **`ai/security-scan`** — diff-aware 的 secret / 依赖漏洞扫描器,用大白话解释发现并发到 PR 评论。
- [ ] **`ai/coverage`** — 对有测试套件的项目,读未测试的源文件,建议补充的测试用例。

### 组织级

- [ ] 给本 `.github` repo 加 CI workflow,按 schedule 跑每个 action 的 `test-*.yml`(跨组织 catch 回归)。
- [ ] 在 repo 根加 `CODE_OF_CONDUCT.md` + `CONTRIBUTING.md`(公开的 `profile/README.md` 暂时够用)。
- [ ] 决定 `ssh` 和 `docker` 是放本组织还是留在 `x-cmd/action` 的 inputs。*(已决定:作为独立 actions 在本组织 ship。`x-cmd/action` 因为向后兼容也保留它们。)*

## 待定问题

- Layer 2 actions 是放 `x-cmd-action` 组织还是别处(比如用户的个人 org)?目前为了可见度公开在 `x-cmd-action`。
- `gitmirror` / `ghwatch` 等要不要共享一个 "config-from-file" 模式(比如 `.config/watched-repos.tsv`)?很可能需要 — 抽一个可复用的 input 约定。
- 命名:`ghwatch` / `ghissuereply` 等是对的吗?要不要改成 `x-cmd-action/watch-github`、`x-cmd-action/reply-issue` 以跟 `gitmirror` 一致?(目前倾向:短、动词在前。)

## 内部参考

- (暂无 — 本文件是内部文档的起点)