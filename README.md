# x-cmd-action (internal)

> **For org maintainers only.** This is the internal view — categorizing actions, tracking planned work, recording design intent. The public-facing version is [`profile/README.md`](./profile/README.md).

## Audience & Purpose

This org builds actions for **super individuals** (超级个体) — people running their own personal workflow pipelines on GitHub Actions. The actions here are designed to be composable, to make a fresh CI runner feel close to a local dev environment, and — increasingly — to **let an AI act as a first-line maintainer** on the user's repo.

We organize actions in three layers. Layers 1 and 2 are about getting the runner wired up correctly (SSH, git config, checkout, monitoring). **Layer 3 is about handing GitHub-native artifacts (issues, PRs, comments, diffs) to an LLM through x-cmd's `x ai request` interface** — auto-labeling new issues, citing the FAQ to answer user questions, posting a draft PR review on every push, extracting post-mortems from closed bugs. The AI is not a demo feature; it's the reason most users adopt this org.

## Layer 1 — Basic Setup

Actions that bring the runner close to a local dev environment. **Composable, mutually independent.** Pick what you need; the goal is "I can use this runner like my laptop."

| Action | Status | Purpose |
| --- | --- | --- |
| [`x-cmd`](https://github.com/x-cmd-action/x-cmd) | ✅ shipped (v1) | install x-cmd into `~/.x-cmd.root/` |
| [`this-repo`](https://github.com/x-cmd-action/this-repo) | ✅ shipped (v1) | minimal clone trigger repo to `$GITHUB_WORKSPACE`. Token-only, no SSH, no extras. 6 inputs. |
| [`ssh`](https://github.com/x-cmd-action/ssh) | ✅ shipped (v1) | ssh-agent + known_hosts + key add |
| [`docker`](https://github.com/x-cmd-action/docker) | ✅ shipped (v1) | docker login + buildx init |
| [`gitconfig`](https://github.com/x-cmd-action/gitconfig) | ✅ shipped (v1) | global git config (name/email + `[include]` for config file). Position-independent — no repo-scoped logic. |

**Why this layer exists.** x-cmd `gitb backup` requires ssh-keyscan. `x gh` requires a GitHub token. Most x-cmd-based actions need *some* of this wiring. Splitting them out lets users compose exactly what they need without paying for what they don't.

**Why `gitconfig` has no `local-config` input.** `local-config` writes to a specific repo's `.git/config`, which implicitly depends on cwd (a specific repo). That's coupling that belongs on the action that *knows* what repo it's checking out — `checkout` / `this-repo`. A global config action should be position-independent. See the FAQ in `gitconfig/README.md`.

## Layer 2 — Common Functions

Self-contained automations built on Layer 1. Each solves one recurring workflow problem.

| Action | Status | Purpose |
| --- | --- | --- |
| [`checkout`](https://github.com/x-cmd-action/checkout) | ✅ shipped (v1) | pure-shell `git checkout`. 22 inputs, 1:1 parity with `actions/checkout@v4` + 3 x-cmd enhancements (`known-hosts-url`, `fetch-additional`, `local-config`). Uses `GIT_SSH_COMMAND` + temp files (mirrors actions/checkout). |
| [`gitmirror`](https://github.com/x-cmd-action/gitmirror) | ✅ shipped (v1) | sync repos you follow across GitHub ↔ Gitee ↔ Codeberg |
| `ghwatch` | 🚧 TODO | watch issues & releases on projects you follow |
| `ghissuereply` | 🚧 TODO | quick-draft replies to incoming issues |
| `ghissuegold` | 🚧 TODO | extract useful patterns / answers from issue threads |
| `webmonitor` | 🚧 TODO | generic URL/diff watcher |
| `hnmonitor` | 🚧 TODO | HN top-stories monitor |

## Layer 3 — AI Assist (built on Layer 1 + 2)

The reason this org adopted x-cmd wasn't only for shell primitives. x-cmd also wraps the entire **AI provider landscape** behind a single stable interface — `x ai request --model <provider>` — and gives every AI action the same dependency-free, pure-shell on-ramp that Layer 1/2 actions already use.

Layer 3 actions are **AI wrappers that operate on GitHub-native artifacts** (issues, PRs, comments, diffs). They are thin: each one reads the artifact via `x gh`, asks `x ai` to transform it, and writes the result back via `x gh`. Same delegation principle as Layer 2 — never reimplement what `x ai request` already does.

| Action | Status | Purpose |
| --- | --- | --- |
| [`ai/triage`](https://github.com/x-cmd-action/ai/triage@v1) | ✅ shipped (v1) | **Auto-route incoming issues.** Reads a new issue's body and existing labels, asks the AI for `type / priority / area / labels / summary`, posts the summary as a comment, and applies the suggested labels. Saves the maintainer the first 5 minutes of every issue. |
| [`ai/reply`](https://github.com/x-cmd-action/ai/reply@v1) | ✅ shipped (v1) | **First-line responder.** Watches for `@x` in issue/comment bodies (strict word-boundary match, configurable). Posts a reaction + canned reply. Combined with a curated FAQ in `README.md` / `docs/`, the bot can cite the FAQ section that answers the question. No AI token required. |
| [`ai/review`](https://github.com/x-cmd-action/ai/review@v1) | ✅ shipped (v1) | **PR review on every push.** Fetches the PR diff via `gh pr diff`, asks the AI for Security / Style / Suggestions / Summary, posts a structured comment on the PR. Cheap enough to run on every PR; accurate enough to catch obvious issues before a human reviews. |
| [`ai/changelog`](https://github.com/x-cmd-action/ai/changelog@v1) | ✅ shipped (v1) | **Weekly community digest.** Runs on `schedule: cron`. Collects issues closed + PRs merged in the last N days, asks the AI to group them into `Features / Fixes / Performance / Docs / Other`, writes `CHANGELOG.md`. Zero-effort weekly update. |
| [`ai/translate`](https://github.com/x-cmd-action/ai/translate@v1) | ✅ shipped (v1) | **i18n on demand.** Reads a Markdown file, translates to a target language, writes the result. Markdown-aware — code blocks preserved, URLs and proper nouns kept. Useful for "I want a Chinese version of my README but I don't speak Chinese" workflows. |
| [`ai/spec`](https://github.com/x-cmd-action/ai/spec@v1) | ✅ shipped (v1) | **RFC + post-mortem drafts.** `mode: rfc` reads a feature request issue and produces a structured RFC (Summary / Motivation / Detailed Design / Alternatives / Drawbacks / Open Questions). `mode: postmortem` extracts timeline + root cause + lessons from a closed bug. Maintainers edit, not write from scratch. |
| [`ai/commit`](https://github.com/x-cmd-action/ai/commit@v1) | ✅ shipped (v1) | **Conventional Commits enforcement.** `mode: check` validates every commit in a PR conforms to the spec, suggests rewrites on failure. `mode: generate` writes a compliant commit message from the staged diff. Pairs with `ai/changelog` — cleaner history, better auto-changelog. |
| [`mneme`](https://github.com/x-cmd-action/mneme) | ✅ shipped (v1) | **AI memory layer.** Persists and retrieves LLM context across workflow runs. Default backend: GitHub Issue (zero cost on public repos). Lets `ai/review` on PR #123 remember what `ai/triage` said on issue #42 (the issue this PR closes) — every Layer 3 action starts from zero context without `mneme`. |

### Concrete use cases (what Layer 3 actually does for users)

The Layer 3 actions are not "AI demos" — each one solves a specific maintainer pain point that today is done manually or not at all.

**1. Auto-label incoming issues** — when a new issue is opened, `ai/triage` reads the body + labels, asks the AI for `type / priority / area / labels / summary`, posts the summary as a comment, and applies the suggested labels. Maintainers stop spending the first 5 minutes of every issue on routing.

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

**2. Read the FAQ and answer user questions** — `ai/reply` watches for `@x` in issue/comment bodies. When triggered, it posts a reaction + reply. Combined with a curated FAQ in `README.md` / `docs/faq.md`, this becomes a cheap first-line responder: the bot cites the FAQ section that answers the question, and the user gets a useful response in seconds instead of waiting for a maintainer.

**3. Auto-investigate incoming issues** — `ai/reply` + a custom prompt can be wired to ask the AI to read the issue + linked repo files + relevant past issues, and post a draft diagnosis ("this looks like the same root cause as #42, fixed in PR #55, line X of file Y"). Maintainers review the draft instead of starting from scratch.

**4. PR code review on every push** — `ai/review` runs on `pull_request: opened / synchronize`, fetches the diff via `gh pr diff`, and posts a structured review (Security / Style / Suggestions / Summary). Cheap enough to run on every PR; accurate enough to catch obvious issues before a human reviews.

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

**5. Conventional Commits enforcement** — `ai/commit` in `mode: check` validates that every commit in a PR conforms to [Conventional Commits](https://www.conventionalcommits.org/). When invalid commits are found, the AI suggests rewrites and posts them as a PR comment. Pairs naturally with `ai/changelog` — the cleaner the commit history, the better the auto-generated changelog.

**6. Weekly community digest** — `ai/changelog` runs on `schedule: cron`, collects issues closed + PRs merged in the last N days, asks the AI to group them into `Features / Fixes / Performance / Docs / Other`, writes the result to `CHANGELOG.md` (or stdout for posting elsewhere). Zero-effort weekly update for users.

**7. RFC + post-mortem extraction** — `ai/spec` in `mode: rfc` reads a feature request issue and produces a structured RFC (Summary / Motivation / Detailed Design / Alternatives / Drawbacks / Open Questions). In `mode: postmortem` it extracts the timeline + root cause + lessons from a closed bug. Maintainers get a draft; they edit, not write from scratch.

**8. i18n on demand** — `ai/translate` reads `README.md`, translates to `target: zh`, writes `README.zh.md`. Markdown-aware — code blocks are preserved, URLs and proper nouns are kept. Useful for "I want a Chinese version of my README but I don't speak Chinese" workflows.

### Why this is a layer, not just "AI features"

- **Stable contract.** All Layer 3 actions share the same `MINIMAX_TOKEN` env convention, the same composite-action shape, and the same `x-cmd-action/x-cmd + x-cmd-action/this-repo` prefix. Users learn one pattern, get seven actions.
- **Provider-agnostic.** `with: model: minimax | openai:gpt-4 | anthropic:claude-fable-5` — switching providers is an input change, not a rewrite. x-cmd's `x ai request` handles the routing.
- **Memory cross-runs.** `mneme` lets `ai/review` on PR #123 remember what `ai/triage` said on issue #42 (the issue this PR closes). Without this, every AI run starts from zero context.
- **Local-first.** Every Layer 3 action corresponds to a local `x ai <subcmd>` — try it on your laptop, ship it as a workflow. Same code path, same output.

**Why `checkout` moved here.** It composes token/SSH/sparse/filter/identity into one action — that's exactly what Layer 2 is for (a recurring workflow job, built on Layer 1). Layer 1 has `this-repo` for users who just want "give me this repo, nothing fancy."

**Why this layer exists.** Common functions can usually be expressed as a Layer 1 setup + an x-cmd script. We package them so users don't pay the maintenance cost of re-discovering the right x-cmd incantation each time. Internally each one is **a thin shell wrapper around x-cmd commands** — never a re-implementation.

## Conventions

- **Pure shell only.** No TypeScript / Node.js bundles.
- **One action per repo.** No monorepos.
- **v1 tag once shipped** + `@main` for the bleeding edge.
- **Apache 2.0** across the org.
- **`profile/README.md`** is the public surface; this file is the maintainer's view.
- **Scope-appropriate naming.** Standalone actions in this org use **unprefixed** input names (`username`, `password`, `ssh-key`, `buildx-init`). The action's own scope is the disambiguator. Prefixes (`docker_username`, `ssh_key`) only appear in `x-cmd/action`, which has 17 inputs spanning multiple domains and needs them to disambiguate. When in doubt: name an input as if it were the only one in scope.

## FAQ

### Why pure shell, not Node.js?

The Node.js-based GitHub Actions ecosystem has a maintenance pattern that's easy to underestimate: actions get "stale" not because their logic changes, but because **Node.js itself is a moving target**. Node 16 → 18 → 20 → 22 → 24 each broke something. `actions/checkout`, `actions/setup-node`, etc. ship regular releases that are mostly "bump the Node version", patch deprecation warnings, fix ESM/CJS interop. Users see constant releases and assume something real changed; the change is usually "this action now runs on Node 24 instead of Node 20".

Pure shell sidesteps this entirely. `bash` is bash. POSIX tools (`git`, `ssh`, `curl`, `grep`, `sed`, `awk`) don't release major versions every six months. A shell action written in 2020 still works on a runner in 2030, because the interpreter and the tools are stable on a timescale measured in decades.

Practical consequences:

- **No version churn.** `v1` means `v1`. We don't bump a Node runtime.
- **Smaller tarball.** ~3 KB shell vs ~few MB bundled JS. Faster checkout, faster failure modes.
- **Trivial to audit.** Read the script, see exactly what runs. No transpiled output, no `node_modules`, no `@actions/*` namespace to learn.
- **Trivial to fork / inline.** Want to customize? Copy the lib shell file into your own action and tweak it.

The trade-off: we don't get to use Node libraries (`@actions/core`, `@actions/github`, `octokit`, …). For our scope — `git`, `ssh`, `docker`, `curl`, `jq` from coreutils — that's fine. When we needed an LLM SDK or a complex HTTP client, we did not pull in a Node dep — we delegated to x-cmd (`x http`, `x curl`, `x ai request`). Layer 3 actions are the proof: they are pure shell that delegate the entire AI call to one `x ai request` line.

The org's principle: **ship shell; lean on x-cmd for anything that would otherwise need a library — including an LLM SDK.**

## Design Principle — Never Reimplement What x-cmd Does

x-cmd ships hundreds of primitives (`x gitb backup`, `x gh`, `x repo`, `x eget`, `x env use`, `x curl`, `x sysinfo`, etc.) that have been iterated on for years. Layer 2 actions in this org **delegate to x-cmd**, they do not reimplement.

### The rule

> An action is a **thin wrapper around x-cmd commands**. It is not a shell reimplementation of x-cmd's logic.

### Concretely

#### ❌ Don't write in shell — let x-cmd do it

| If you need to... | Don't write... | Use x-cmd instead |
| --- | --- | --- |
| Mirror a repo | `git clone --bare` + `git push` | `x gitb backup` |
| Call GitHub API | custom REST client | `x gh` |
| Fetch a URL | custom HTTP wrapper | `x curl` / `x http` / `x fetch` |
| Parse JSON | custom parser | `x jq` / `x json` |
| Install a language toolchain | `apt install` / brew scripts | `x env use <lang>` |
| Install a binary | curl + chmod + PATH hack | `x eget use` |

#### ✅ Do write in shell — but only for action glue

- Parse action inputs into a structure x-cmd understands
- Wire up secrets / credentials (this is the Layer 1 actions' job)
- Call x-cmd with the parsed inputs
- Format x-cmd output into GitHub-flavored outputs (markdown, `$GITHUB_OUTPUT`, `$GITHUB_STEP_SUMMARY`)
- Implement GitHub Actions–specific glue (matrix, conditionals, env files)

### Exceptions (where shell reimplementation is OK)

**Just one: GitHub Actions–native glue.** x-cmd lives outside the GitHub Actions runtime, so it can't write to `$GITHUB_OUTPUT`, render `$GITHUB_STEP_SUMMARY`, evaluate `if:` expressions, declare matrix strategies, or read secrets. That orchestration code is the action's job — write it in shell.

**Everything else, call x-cmd.** Apply the heuristic: *if the process can be atomized and clearly described, it should be expressible as one or more x-cmd commands.* So:

- "Make an HTTPS request to LLM X" is atomizable → `x curl` does it. Not an exception.
- "Persist key/value state across steps" is atomizable → x-cmd has file/env primitives. Not an exception.
- "Write to `$GITHUB_OUTPUT`" is **not** atomizable into x-cmd — it's a GitHub Actions runtime concern. Exception.

When in doubt: write the x-cmd version first. If x-cmd genuinely can't do it, the exception applies.

### Why this matters

| Aspect | Reimplementing | Delegating to x-cmd |
| --- | --- | --- |
| Code size | 5+ lines per primitive | 1 line per call |
| Bug fixes | You maintain it | x-cmd maintainer fixes; all actions benefit |
| Performance | Whatever you write | Whatever x-cmd is tuned to |
| Behavior parity | Diverges from `x gitb backup` etc. | 100% matches local `x ...` |
| Tests | You write them | x-cmd's tests cover it |

### Anti-pattern example (gitmirror v0)

```bash
# WRONG — reimplemented what x gitb backup already does
git clone --bare "$src" "$work/repo.git"
cd "$work/repo.git"
git remote add target "$dest"
git push target --all --tags --force
```

### Right pattern (gitmirror v1)

```bash
# RIGHT — one line, delegates
x gitb backup --force "$src" "$dest"
```

## TODO

### Extract from `x-cmd/action`

- [x] **`x-cmd-action/ssh`** — extract the ssh-agent / known_hosts setup from `x-cmd/action` into a standalone action (input: `ssh_key`). Lets users add SSH to any workflow without pulling in the full `x-cmd/action`. — **Done, v1 shipped.**
- [x] **`x-cmd-action/docker`** — extract `docker login` + `docker buildx create --use` from `x-cmd/action` into a standalone action (inputs: `docker_username`, `docker_password`, `docker_buildx_init`). — **Done, v1 shipped.**

### New Layer 2 actions

- [ ] **`x-cmd-action/ghwatch`** — `on: schedule: cron: '0 */6 * * *'`. For a list of `repo` in input, fetch recent issues + releases via `x gh`. Output as JSON to `$GITHUB_STEP_SUMMARY` or push to a gist.
- [ ] **`x-cmd-action/ghissuereply`** — given an issue number, draft a reply using a configurable template + LLM call. Use `x gh` for fetching, output the draft as a PR-able markdown file.
- [ ] **`x-cmd-action/ghissuegold`** — given a closed issue (or thread), summarize the accepted answer / fix into a reusable snippet. Output to a knowledge-base file via PR.
- [ ] **`x-cmd-action/webmonitor`** — generic: take a list of URLs, fetch, diff against cached snapshot, open an issue if changed.
- [ ] **`x-cmd-action/hnmonitor`** — Hacker News top stories monitor. Uses `x web` or `x curl` to hit the HN API, output new stories as a digest.

### Layer 3 (AI) — done

- [x] **`x-cmd-action/ai`** — monorepo with seven sub-commands (`triage`, `reply`, `review`, `changelog`, `translate`, `spec`, `commit`), each mapping 1:1 to local `x ai <subcmd>`. — **Done, v1 shipped.**
- [x] **`x-cmd-action/mneme`** — AI memory layer (store / retrieve / search across workflow runs; default backend: GitHub Issue). — **Done, v1 shipped.**

### Future Layer 3 ideas

- [ ] **`ai/rank`** — rank incoming issues by "user pain" estimate (bug frequency × severity × user impact) so the maintainer triages by urgency, not arrival order.
- [ ] **`ai/reply-context`** — pull relevant snippets from `docs/` / `README.md` / past closed issues, then have the bot cite them when answering. Improves over plain `ai/reply` for repos with rich docs.
- [ ] **`ai/security-scan`** — diff-aware secret / dependency-vuln scanner that explains findings in plain language and posts as PR comment.
- [ ] **`ai/coverage`** — for projects with a test suite, suggest missing test cases by reading untested source files.

### Org-level

- [ ] Add CI workflow to this `.github` repo that runs every action's `test-*.yml` on schedule (catch regressions across the org).
- [ ] Add `CODE_OF_CONDUCT.md` + `CONTRIBUTING.md` at the repo root (the public-facing `profile/README.md` is enough for now).
- [ ] Decide whether `ssh` and `docker` go in this org or stay as inputs of `x-cmd/action`. *(Done: shipped as standalone actions in this org. `x-cmd/action` still has them for backward compat.)*

## Open questions

- Should Layer 2 actions live in `x-cmd-action` org or somewhere else (e.g., the user's personal org)? Currently public in `x-cmd-action` for visibility.
- Should `gitmirror` / `ghwatch` etc. share a common "config-from-file" pattern (e.g., `.config/watched-repos.tsv`)? Probably yes — extract a reusable input convention.
- Naming: are `ghwatch` / `ghissuereply` etc. the right names? Or rename to `x-cmd-action/watch-github`, `x-cmd-action/reply-issue` for consistency with `gitmirror`? (Current preference: short, verb-first.)

## Related internal docs

- (none yet — this file is the start of internal documentation)