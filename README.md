# x-cmd-action (internal)

> **For org maintainers only.** This is the internal view — categorizing actions, tracking planned work, recording design intent. The public-facing version is [`profile/README.md`](./profile/README.md).

## Audience & Purpose

This org builds actions for **super individuals** (超级个体) — people running their own personal workflow pipelines on GitHub Actions. The actions here are designed to be composable and to make a fresh CI runner feel close to a local dev environment, so users can iterate on personal automation cheaply.

We organize actions in two layers.

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

The trade-off: we don't get to use Node libraries (`@actions/core`, `@actions/github`, `octokit`, …). For our scope — `git`, `ssh`, `docker`, `curl`, `jq` from coreutils — that's fine. If we ever needed an LLM SDK or a complex HTTP client, we'd delegate to x-cmd (`x http`, `x curl`) rather than pull in a Node dep.

The org's principle: **ship shell; lean on x-cmd for anything that would otherwise need a library.**

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