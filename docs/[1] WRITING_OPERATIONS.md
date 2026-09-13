# Writing Operations — Converting Shell Commands into MCP Operations

> **Companion to `[0] MCP_SERVER_DEVVM.md`.** That document says *why* the agent gets typed operations instead of a shell. This one is the practical guide to actually converting your commands, with worked examples, a review checklist, and a starter set you can copy.
> **If you read one section, read §1.** Almost every bad operation comes from getting that wrong.

---

## 0. The one rule

> **You are not converting commands. You are converting intents.**

A shell command is one *implementation* of something a person wanted. Convert the command and you get a shell with extra syntax. Convert the want and you get an operation.

```
❌  "We run `npm run test:unit`"      →  npm.run(script: str)
✅  "We run the unit tests"           →  test.run(suite: enum[unit, integration, e2e])
```

The first one looks like progress and is not: `script` selects behaviour from `package.json`, which is a file in the workspace the agent can edit. You have built a shell that requires two approvals.

Everything below is elaboration on that one rule.

---

## 1. Altitude — the judgment call that decides everything

Every conversion has an altitude: how much of the task the operation covers.

```
TOO LOW          npm.run(script)          a shell with a prefix
                 git.raw(args)            a shell with a different prefix
                 file.write(path, body)   a shell, eventually

RIGHT            test.run(suite)          a named task, narrow effect
                 git.checkout(repo, branch)
                 service.restart(unit)

TOO HIGH         deploy.everything()      unreviewable, unscopable
                 fix.the.build()          effect unbounded by construction
```

### 1.1 Two tests for correct altitude

**Test 1 — the one-sentence test.** Can you write a single sentence that is both **true** and **specific** about what this does?

```
✅  "Runs the unit test suite in the api workspace and returns pass/fail counts."
❌  "Runs an npm script, which depending on package.json may do anything."
❌  "Deploys."
```

If your sentence needs "depending on the arguments" or "may", you are too low. If it has no object, you are too high.

**Test 2 — the reviewer test.** Could a competent reviewer approve this from the approval prompt **alone**, without opening a terminal to check current state?

Approving `npm.run("build")` requires knowing what `build` does in `package.json` *right now*. The reviewer can't know that from the prompt, so the prompt is a lie by omission. Approving `test.run(unit)` requires nothing external.

This is the practical form of the architecture doc's central claim: the human approves an effect, not a string. **An operation whose effect depends on mutable state the reviewer can't see has failed before you write a line of code.**

### 1.2 Finding the right altitude

Ask: *what was the person trying to accomplish?* Keep asking until the answer stops being about tools.

```
"run npm run test:unit"                      ← a command
  ↓ why?
"run the unit tests"                         ← a task          ← STOP HERE
  ↓ why?
"find out if my change broke anything"       ← a goal (too high for one operation)
```

The task layer is your operation. It is the layer where the name means something to a person, the effect is bounded, and the parameters are a small closed set.

### 1.3 When one intent needs several operations

Splitting is usually right. Combining is usually wrong.

```
✅  git.checkout, git.pull, git.status              three intents, three operations
❌  git.do(action: enum[checkout, pull, status])    one operation, three effects,
                                                      one effect declaration that must
                                                      be the union of all three
```

The second form breaks §5.2 of the architecture doc: the effect declaration becomes the *superset* of every branch, so the sandbox is wider than any single call needs and the approval text is vague. **One operation, one effect.** If the declarations differ, they're different operations.

---

## 2. The method

Eight steps. Steps 1–5 are design and take the time; 6–8 are mechanical.

```
1. Name the intent                    →  operation id + summary
2. Triage the varying parts           →  params
3. Determine what it touches          →  effect declaration
4. Determine what must be true first  →  preconditions
5. Determine what could go wrong      →  risk class + approval + reversibility
6. Implement without a shell          →  the Python
7. Return structured data             →  the result schema
8. Prove the declaration              →  conformance test
```

Work them in order. Steps 3 and 4 frequently send you back to step 1 — if the effect declaration is sprawling or you can't state a precondition, your altitude is wrong.

---

## 3. Step 1 — Name the intent

`domain.verb`, lowercase, snake_case within segments.

```
✅  git.checkout      test.run          service.restart     logs.tail
    deps.install      db.migrate        build.container     cache.clear

❌  run_command       exec              do_thing            git_helper
    utils.generic     admin.action      misc.run
```

Rules:

- **The domain is a bounded area of the system**, not a binary. `test`, not `npm`. `service`, not `systemctl`. Naming after the tool locks you to the tool and tempts you toward `tool.raw(args)`.
- **The verb is what happens**, not how. `restart`, not `run_systemctl`.
- **No `generic`, `misc`, `util`, `helper`, `raw`, `exec`, `custom`, `advanced`.** Every one of these is a smell that you're building an escape hatch with a respectable name.
- The summary is one sentence, present tense, with an object: *"Checks out a branch in a workspace repository."*

---

## 4. Step 2 — Parameter triage

For every part of the command that varies, pick exactly one of four:

| | Use when | Example |
|---|---|---|
| **Fixed** | It doesn't really vary | `--prune` on a fetch |
| **Enum** | The valid set is knowable at config time | `repo`, `service`, `environment`, `suite` |
| **Validated scalar** | Open set, but constrained shape | `branch`, `line_count`, `commit_sha` |
| **Never a parameter** | It's a path, URL, flag, command, or script name | — |

### 4.1 Default to fixed

Most of a command doesn't need to vary. Bake it in.

```python
await wt.git("fetch", "origin", "--prune")     # ✅ no parameters at all
```

Every parameter is a decision you've handed to the agent. Hand over as few as possible.

### 4.2 Prefer enums, aggressively

**If the valid set is knowable when you write the config, it is an enum.** This covers far more than people expect:

```yaml
repo:        {type: enum, values: [api, web, infra]}
service:     {type: enum, values: [api, worker, scheduler]}
environment: {type: enum, values: [dev, staging]}
suite:       {type: enum, values: [unit, integration, e2e]}
log_stream:  {type: enum, values: [api, worker, nginx, system]}
```

Enums give you four things for free: no injection surface, a manifest that documents itself, an approval prompt that reads in English, and a compile-time list of everything reachable.

**Paths are always enums** — or rather, path *keys* are. The agent names `api`; the daemon resolves it from config:

```python
root = config.workspaces[repo]       # "api" → /srv/workspace/api
```

The agent never sees or supplies a path. This kills traversal as a category rather than defending against it.

### 4.3 Validated scalars, when the set is genuinely open

```yaml
branch:
  type: string
  pattern: '^[A-Za-z0-9._/-]{1,100}$'
  max_length: 100
  no_leading_dash: true            # ← never omit this

lines:
  type: integer
  min: 1
  max: 1000

commit_sha:
  type: string
  pattern: '^[a-f0-9]{7,40}$'
```

Rules for every scalar:

```
□ Anchored fullmatch pattern, never a search
□ Explicit max_length
□ no_leading_dash unless you have a specific, documented reason
□ Control characters (NUL, CR, LF) rejected
□ Unicode NFC-normalized; bidi and zero-width flagged
□ Numeric parameters are integers with bounds, never free-form strings
```

`no_leading_dash` is the one people skip. `branch = "-c core.pager=sh"` is arbitrary execution through a perfectly well-formed argv list — the pattern above already rejects it (no spaces, no `=`), but defence in depth is the point, and a laxer pattern elsewhere will eventually not.

### 4.4 Never parameters

```
☠️  A path                  → use an enum key resolved server-side
☠️  A URL                   → use a named, configured endpoint
☠️  A flag or option        → if two behaviours are needed, that's two operations
                              or a mode: enum
☠️  A command or fragment   → this is the escape hatch; see §11
☠️  A script/target name    → see §10, the indirection trap
☠️  A timeout               → comes from the manifest; an agent-set timeout is a DoS knob
☠️  A sandbox or env setting → derived from the manifest, never from the request
```

**Flags deserve emphasis.** Shell users think in flags, so the instinct is `git.checkout(branch, flags: list[str])`. Flags change behaviour, which means the effect declaration can no longer be accurate, which means the approval prompt can no longer be true. If you need `--force`, that is `git.checkout_force`, a different operation with a different risk class — and it *should* feel heavier to reach for.

### 4.5 Mode enums, when two variants really are one intent

```yaml
# ✅ acceptable: closed set, each mode individually reviewable
mode: {type: enum, values: [soft, hard]}
```

Acceptable when the modes share an effect declaration and the *union* is still narrow. If `hard` writes somewhere `soft` doesn't, split them — the declaration must be true for every call, not on average.

---

## 5. Step 3 — Declare the effect

This is both the approval text and the sandbox policy (architecture doc §5.2), so being lazy here is expensive in two directions at once.

```yaml
effect:
  description: |
    Fetches from origin and checks out {branch} in the {repo} workspace.
    Leaves untracked files alone. Does not push, reset, or delete anything.
  writes:   ["/srv/workspace/{repo}"]
  reads:    ["/srv/workspace/{repo}"]
  network:  ["git.internal.example:443"]
  services: []
```

**How to get it right:** run the command under observation and look at what it actually touched.

```bash
strace -f -e trace=openat,connect,execve -o /tmp/trace \
  git -C /srv/workspace/api checkout feature/x
awk '/openat|connect/' /tmp/trace | grep -v ENOENT | sort -u
```

You will be surprised at least once. Git reads `/etc/gitconfig` and `~/.gitconfig`. Node reaches for a proxy env var. A test runner writes a cache directory nobody mentioned. **Find those now, in a terminal, rather than as a sandbox failure in production.**

Rules:

```
□ Narrowest paths that work. Not /srv/workspace — /srv/workspace/{repo}.
□ reads and writes are separate. A read-only mount is free; use it.
□ network defaults to [] — no network at all, kernel-enforced.
□ Never 0.0.0.0/0. If it seems necessary, that's a design conversation.
□ Parameters may appear in paths ({repo}) only when the parameter is an enum,
  so the expansion set is finite and reviewable.
□ The description says what it does NOT do, where a reviewer might assume otherwise.
```

That last line is worth the effort. *"Leaves untracked files alone. Does not push."* pre-empts exactly the question a careful reviewer has, and it makes the diff meaningful when someone later changes it.

---

## 6. Step 4 — Preconditions

A precondition is a fact captured at proposal time, shown to the human, and re-verified adjacent to the effect. It's what stops an approval being applied to state that has since changed.

```yaml
preconditions:
  - git.head_sha
  - git.worktree_clean
```

```python
ctx.require(Precondition("git.head_sha",       await wt.head_sha()))
ctx.require(Precondition("git.worktree_clean", await wt.is_clean()))
```

### 6.1 Choosing them

Ask: **what would make this operation do something different, or something destructive, if it changed between approval and execution?**

| Operation family | Preconditions |
|---|---|
| Git mutations | `head_sha`, `worktree_clean`, `current_branch` |
| File mutations | content hash of the target, or its absence |
| Service control | `is_active`, unit fragment path |
| Container ops | image digest, container id |
| Dependency install | lockfile hash |
| Database | migration version |

### 6.2 Rules

- **`DESTRUCTIVE` operations must have at least one.** If you can't define one, that's a strong signal the operation shouldn't exist in this form.
- **Check adjacent to the effect**, in the same function, not a round trip away.
- **Use a genuinely atomic form where one exists** — `git update-ref` with an old-value, `O_EXCL` create, `flock`, a conditional DB update. Then the window is zero rather than small.
- **Read-only operations usually need none.** Don't add ceremony where there's nothing to protect.
- Preconditions appear in the approval prompt, so keep them human-readable. `head_sha: a1b2c3d4` is useful; `state_blob: <4KB of JSON>` is not.

---

## 7. Step 5 — Risk, approval, reversibility

```yaml
risk: MUTATING_LOCAL
approval: required
reversible:
  kind: git_ref
  restore_hint: "git checkout {preconditions.git.head_sha}"
```

### 7.1 Picking the class

```
READ_ONLY        no writes, no side effects, no network egress beyond a declared read
MUTATING_LOCAL   writes within a declared workspace root
SERVICE_CONTROL  starts/stops/restarts a declared unit
NETWORK_EXEC     downloads and runs third-party code — installs, builds
DESTRUCTIVE      deletes data, force-pushes, prunes, resets hard, drops
PRIVILEGE        needs elevation — not grantable through this server
```

**Two traps:**

**`NETWORK_EXEC` is under-used.** `npm install`, `pip install`, `cargo build`, `docker build`, `make` all fetch and execute code you didn't write. They feel like build steps. They are "run arbitrary code from the internet," and they belong in an ephemeral container (architecture doc §14.3). If your operation's `network` list is non-empty **and** it executes anything it fetched, it is `NETWORK_EXEC`.

**`READ_ONLY` is over-used.** It must be *genuinely* side-effect-free. `git status` can write `.git/index`. `docker ps` is fine. A test run that populates a cache is not read-only. If `writes` is non-empty, it isn't `READ_ONLY`.

### 7.2 Reversibility

Be honest — this appears in the approval prompt and reviewers act on it.

```yaml
reversible: {kind: git_ref, restore_hint: "git checkout a1b2c3d4"}   # yes, precisely
reversible: {kind: snapshot_required}                                 # only with a snapshot
reversible: {kind: none, note: "Deleted files are not recoverable."}  # say so plainly
```

`kind: none` on a `DESTRUCTIVE` operation should trigger "does this need a pre-run snapshot?" in review — often the answer is yes, and the platform makes it cheap.

---

## 8. Step 6 — Implement without a shell

### 8.1 The hierarchy

**Best — a library. No subprocess at all.**

```python
import pygit2
repo = pygit2.Repository(root)
repo.checkout(f"refs/heads/{branch}")
```
No process, no argv, no PATH, no environment. Structured errors. Fastest.

**Good — argv list through the framework.**

```python
await wt.git("checkout", "--", branch)
```
No shell. Each element is one argument. `--` terminates option parsing.

**Never — a string.**

```python
subprocess.run(f"git checkout {branch}", shell=True)     # ☠️
subprocess.run(" ".join(["git", "checkout", branch]))    # ☠️ same thing, worse disguise
os.system(f"git checkout {branch}")                      # ☠️
```

The CI gate in the architecture doc (§3.3) fails the build on all three. Keep it that way.

### 8.2 Implementation checklist

```
□ No shell=True, no os.system, no string-built commands
□ argv elements are separate list items — a parameter never joins into an element
□ Paths come from config lookup, never from a parameter
□ Environment is constructed, not inherited (no os.environ passthrough)
□ Timeout comes from the manifest
□ Output bounded while reading, not trimmed after
□ Errors are typed exceptions, never raw stderr relayed onward
□ Nothing touched that the manifest doesn't declare
□ Idempotent where the underlying operation allows it
```

### 8.3 Environment

```python
# ☠️  inherits PATH, LD_PRELOAD, proxies, tokens, anything the agent planted earlier
env = os.environ.copy()

# ✅  constructed from nothing
env = {
    "PATH": "/usr/local/bin:/usr/bin:/bin",
    "HOME": str(sandbox_home),
    "LANG": "C.UTF-8",
    "GIT_TERMINAL_PROMPT": "0",        # never block waiting for credentials
    "GIT_CONFIG_NOSYSTEM": "1",        # ignore /etc/gitconfig
}
```

Declare any additional variable in the manifest. `GIT_TERMINAL_PROMPT=0` is the kind of detail that turns a mysterious timeout into a clean error.

---

## 9. Step 7 — Return structured data

This is the most under-appreciated win in the whole conversion, and it costs almost nothing.

```python
# ❌ dumping stdout — the agent must parse it, and it's all untrusted text
return Result(stdout=await wt.git("status", "--porcelain"))

# ✅ structured, typed, small
return Result(
    summary=f"On {branch}, {len(changed)} changed files",
    data={
        "branch": branch,
        "head_sha": sha,
        "clean": not changed,
        "changed_files": changed[:50],
        "ahead": ahead, "behind": behind,
    },
)
```

Four benefits:

1. **The agent doesn't parse text**, so it doesn't misparse text.
2. **Far less context consumed.** A structured status is ~200 tokens; `git status -vv` output can be thousands. Context is the scarcest resource in an agent loop.
3. **Less injection surface.** Typed fields in a known schema are a much smaller target than arbitrary text — and a branch name in a `branch` field reads as data, not as instruction.
4. **The schema is a contract.** You can change the implementation without breaking the agent.

Declare it:

```yaml
returns:
  branch:        {type: string}
  head_sha:      {type: string}
  clean:         {type: boolean}
  changed_files: {type: array, items: string, max_items: 50}
  ahead:         {type: integer}
  behind:        {type: integer}
```

**Cap every collection.** `changed_files` on a repo with 40,000 modified files must not become 40,000 entries in the agent's context. Truncate and set a `truncated: true` flag.

Keep raw output available for the genuinely unstructured cases (a test failure trace, a build log) — but as *paged, bounded, labelled* output the agent fetches deliberately, not as the default return value.

---

## 10. ⭐ The indirection trap

The single most common way a good-looking operation turns out to be a shell.

> **If a parameter selects behaviour from a file the agent can write, that parameter is arbitrary code execution.**

Every one of these is the same bug:

| Operation | The file | What the agent does |
|---|---|---|
| `npm.run(script)` | `package.json` | Adds a `"pwn": "curl … \| sh"` script, then runs it |
| `make.target(name)` | `Makefile` | Adds a target |
| `docker.compose_up(service)` | `docker-compose.yml` | Adds a service with `/:/host` mounted |
| `git.run_alias(alias)` | `.git/config` | `alias.x = !sh` |
| `service.restart(unit)` with a writable unit dir | `*.service` | Writes a unit, restarts it as root |
| `test.run(config_file)` | the config | `conftest.py`, a custom reporter, a plugin |
| `script.run(name)` from a writable dir | the script | Writes the script |

It takes **two** approvals — one to write the file, one to run it — and the second approval looks completely innocuous, because `npm.run("build")` *is* innocuous right up until `build` isn't what it was.

### 10.1 Three fixes, in order of preference

**1. Bake the behaviour into the operation.** Best when there are a few known variants.

```python
@operation("test.run", version=1)
async def test_run(ctx, repo: str, suite: str) -> Result:
    argv = {                                          # the mapping lives HERE, in reviewed code
        "unit":        ["npx", "jest", "--ci", "--selectProjects", "unit"],
        "integration": ["npx", "jest", "--ci", "--selectProjects", "integration"],
        "e2e":         ["npx", "playwright", "test", "--reporter=json"],
    }[suite]
    return await ctx.workspace(repo).run(argv)
```

**2. Make the file immutable to the agent.** If the indirection must stay, remove the write.

```
□ package.json / Makefile / compose files owned by another account, mode 0444
□ No operation declares them in `writes`
□ Changing them is a deploy, not a runtime action
```

**3. Pin the file's hash as a precondition.** Weakest, but sometimes the practical answer for a file that legitimately changes with the code.

```yaml
preconditions: [package_json.sha256]
```

Now the approval is bound to a specific `package.json`, and editing it invalidates every in-flight approval. The reviewer still can't see what's in it — so use this only alongside option 2 or when the file is code-reviewed on its way in.

### 10.2 The general test

For every parameter, ask: **can the agent change what this value means, without changing the value?**

If yes, it's indirection, and you need one of the three fixes.

---

## 11. Worked conversions

Five, in increasing difficulty. The fourth and fifth are where real runbooks live.

### 11.1 Easy — `git status`

**The command:** `git -C /srv/workspace/api status --porcelain -b`
**The intent:** *"What state is the api repo in?"*

```yaml
id: git.status
version: 1
summary: "Report the working-tree state of a workspace repository"
params:
  repo: {type: enum, values: [api, web, infra]}
effect:
  description: "Reads the git state of the {repo} workspace. Changes nothing."
  reads:   ["/srv/workspace/{repo}"]
  writes:  []
  network: []
risk: READ_ONLY
approval: not_required
returns:
  branch:        {type: string}
  head_sha:      {type: string}
  clean:         {type: boolean}
  ahead:         {type: integer}
  behind:        {type: integer}
  changed_files: {type: array, items: string, max_items: 50}
  truncated:     {type: boolean}
sandbox: {mode: namespace, timeout_s: 15}
```

```python
@operation("git.status", version=1)
async def git_status(ctx, repo: str) -> Result:
    wt = ctx.workspace(repo)
    porcelain = await wt.git("status", "--porcelain=v1", "-b")
    parsed = parse_porcelain(porcelain)              # tested separately
    files = parsed.files[:50]
    return Result(
        summary=f"{parsed.branch} — {'clean' if parsed.clean else f'{len(parsed.files)} changed'}",
        data={**parsed.header(), "changed_files": files,
              "truncated": len(parsed.files) > 50},
    )
```

Note what happened: `--porcelain` was chosen by *you* because it's stable to parse, not exposed as a format parameter. And the return is a typed object, not the porcelain text.

### 11.2 Parameters — `git checkout`

**The command:** `git -C /srv/workspace/api fetch origin --prune && git checkout feature/x`
**The intent:** *"Switch the api repo to branch X."*

Parameter triage:

| Part | Decision | Why |
|---|---|---|
| `-C /srv/workspace/api` | **enum** `repo` | Set knowable at config time |
| `fetch origin --prune` | **fixed** | Always wanted; not a decision for the agent |
| `feature/x` | **validated scalar** `branch` | Open set, constrained shape |
| `-f` / `--force` | **not a parameter** | Different risk class ⇒ a different operation if ever needed |

```yaml
id: git.checkout
version: 3
summary: "Check out a branch in a workspace repository"
params:
  repo:   {type: enum, values: [api, web, infra]}
  branch: {type: string, pattern: '^[A-Za-z0-9._/-]{1,100}$', max_length: 100,
           no_leading_dash: true}
effect:
  description: |
    Fetches from origin and checks out {branch} in the {repo} workspace.
    Leaves untracked files alone. Does not push, reset, or delete anything.
  reads:   ["/srv/workspace/{repo}"]
  writes:  ["/srv/workspace/{repo}"]
  network: ["git.internal.example:443"]
risk: MUTATING_LOCAL
approval: required
preconditions: [git.head_sha, git.worktree_clean]
reversible: {kind: git_ref, restore_hint: "git checkout {preconditions.git.head_sha}"}
sandbox: {mode: namespace, timeout_s: 120}
```

```python
@operation("git.checkout", version=3)
async def git_checkout(ctx, repo: str, branch: str) -> Result:
    wt = ctx.workspace(repo)
    ctx.require(Precondition("git.head_sha",       await wt.head_sha()))
    ctx.require(Precondition("git.worktree_clean", await wt.is_clean()))

    await wt.git("fetch", "origin", "--prune")
    await wt.git("checkout", "--", branch)          # -- terminates option parsing

    return Result(summary=f"Checked out {branch} in {repo}",
                  data={"branch": branch, "head_sha": await wt.head_sha()})
```

`worktree_clean` as a precondition is a design decision worth noticing: it means the operation refuses when there are uncommitted changes, rather than silently carrying or stashing them. That is the right default — the alternative surprises people — and it's exactly the kind of thing the reviewer sees in the prompt.

### 11.3 The trap — `npm run <script>`

**The command:** `npm run test:unit`
**The naive conversion:** `npm.run(script: str)` — and now you have §10.

**The right question:** *what do people actually run?* Look at the last 200 invocations. It's almost always three or four scripts.

```yaml
id: test.run
version: 1
summary: "Run a named test suite in a workspace repository"
params:
  repo:  {type: enum, values: [api, web]}
  suite: {type: enum, values: [unit, integration, e2e]}
effect:
  description: |
    Runs the {suite} test suite in {repo}. Writes only to the test cache and
    report directories. Reaches no network.
  reads:   ["/srv/workspace/{repo}"]
  writes:  ["/srv/workspace/{repo}/.cache", "/srv/workspace/{repo}/reports"]
  network: []
risk: MUTATING_LOCAL       # NOT read-only — it writes a cache
approval: required
returns:
  passed: {type: integer}
  failed: {type: integer}
  duration_s: {type: number}
  failures: {type: array, items: object, max_items: 20}
sandbox: {mode: namespace, timeout_s: 900}
```

```python
SUITES = {                                       # the mapping lives in reviewed code
    "unit":        ["npx", "jest", "--ci", "--json", "--selectProjects", "unit"],
    "integration": ["npx", "jest", "--ci", "--json", "--selectProjects", "integration"],
    "e2e":         ["npx", "playwright", "test", "--reporter=json"],
}

@operation("test.run", version=1)
async def test_run(ctx, repo: str, suite: str) -> Result:
    out = await ctx.workspace(repo).run(SUITES[suite], allow_nonzero=True)
    report = json.loads(out.stdout)              # --json, so we parse structure not text
    return Result(
        summary=f"{report['numPassedTests']} passed, {report['numFailedTests']} failed",
        data={"passed": report["numPassedTests"], "failed": report["numFailedTests"],
              "duration_s": report["duration"] / 1000,
              "failures": summarize_failures(report)[:20]},
    )
```

Three things this gets right that the naive version doesn't: the script mapping is reviewed code rather than a workspace file; `network: []` means a test that tries to phone home fails loudly; and `allow_nonzero=True` treats a failing suite as a **normal result**, not an error — a failing test is data, not an incident.

### 11.4 Multi-step — a real runbook line

**The runbook:**

```
ssh dev-vm
cd /srv/workspace/api
git pull
npm ci
npm run migrate
sudo systemctl restart api
curl -f localhost:8080/health
```

**The wrong conversion:** six operations the agent chains. That gives you six approvals, and — worse — the *composition* is never reviewed. Nobody approved "pull, then migrate, then restart"; they approved six fragments and the agent decided the order. If step 3 fails, the agent decides what to do next, unsupervised, with the system half-updated.

**The right conversion:** ask what this *is*. It's "update the dev API to the latest main." That is one intent.

```yaml
id: api.refresh_dev
version: 1
summary: "Update the dev API service to the latest main and restart it"
params:
  skip_migrations: {type: boolean, default: false}
effect:
  description: |
    Pulls main in the api workspace, installs locked dependencies, runs pending
    database migrations against the DEV database, restarts the api service, and
    waits for its health check. Does not touch staging or production. Rolls the
    service back to the previous commit if the health check fails.
  reads:   ["/srv/workspace/api"]
  writes:  ["/srv/workspace/api", "/var/lib/devdb"]
  network: ["git.internal.example:443", "registry.npmjs.org:443"]
  services: ["api.service"]
risk: NETWORK_EXEC          # npm ci runs third-party install scripts
approval: required
preconditions: [git.head_sha, git.worktree_clean, package_lock.sha256, db.migration_version]
reversible: {kind: scripted, restore_hint: "Automatic rollback to {git.head_sha} on failure"}
sandbox: {mode: container, image: "registry.internal/devd-node:20@sha256:…", timeout_s: 900}
```

```python
@operation("api.refresh_dev", version=1)
async def api_refresh_dev(ctx, skip_migrations: bool = False) -> Result:
    wt = ctx.workspace("api")
    before = await wt.head_sha()
    ctx.require(Precondition("git.head_sha", before))
    ctx.require(Precondition("git.worktree_clean", await wt.is_clean()))

    steps: list[str] = []
    try:
        await wt.git("pull", "--ff-only", "origin", "main")
        steps.append("pulled")

        await wt.run(["npm", "ci"])                      # ci, not install: lockfile-exact
        steps.append("deps")

        if not skip_migrations:
            await wt.run(["npx", "prisma", "migrate", "deploy"])
            steps.append("migrated")

        await ctx.service("api.service").restart()       # narrow privileged helper
        steps.append("restarted")

        await ctx.http_probe("http://127.0.0.1:8080/health", timeout_s=30)
        steps.append("healthy")

    except OperationError as exc:
        # The rollback is IN the operation — reviewed, tested, and not the agent's judgment.
        await wt.git("reset", "--hard", before)
        await ctx.service("api.service").restart()
        return Result(summary=f"Failed at {steps[-1] if steps else 'start'}; rolled back to {before[:8]}",
                      data={"ok": False, "completed": steps, "rolled_back_to": before},
                      failed=True)

    return Result(summary=f"Dev API updated to {(await wt.head_sha())[:8]}",
                  data={"ok": True, "completed": steps, "head_sha": await wt.head_sha()})
```

What you gained by combining here rather than splitting:

- **One approval, for the thing that actually happens**, with an effect description a reviewer can evaluate as a whole.
- **The failure path is reviewed code**, not agent improvisation at 2am.
- **The sequence is atomic-ish** — no window for the agent to do something else between migrate and restart.
- **Preconditions cover the whole sequence**, so a concurrent change invalidates the whole thing rather than half of it.

This looks like it contradicts §1.3's "splitting is usually right." It doesn't: **split by intent, not by command.** Six commands, one intent. The test is still §1.1 — one true, specific sentence, which this has.

### 11.5 Hard — "tail the logs and figure out what's failing"

This one has no single operation, and pretending otherwise is how `logs.grep(pattern)` gets born and then grows a `--exec` flag.

**Decompose into read-only operations plus agent judgment:**

```yaml
id: logs.tail
params:
  stream: {type: enum, values: [api, worker, nginx, system]}
  lines:  {type: integer, min: 1, max: 1000, default: 200}
  since:  {type: enum, values: [5m, 15m, 1h, 6h, 24h], default: 15m}
  level:  {type: enum, values: [any, warn, error], default: any}
risk: READ_ONLY
returns:
  entries: {type: array, items: object, max_items: 1000}   # parsed, structured
  truncated: {type: boolean}
  counts_by_level: {type: object}
```

Plus `service.status(unit)`, `health.check(service)`, `metrics.recent(service, window)`.

The agent composes *reads* — which is exactly what it's good at — and any resulting *action* is a separate operation with its own approval. Note `level` and `since` as enums rather than a free-form `grep` pattern: it covers the real need, and it does not hand over a regex engine.

**When the agent genuinely needs something you don't have**, that's the coverage signal (architecture doc §5.5): it lands in the `OPERATION_UNAVAILABLE` log, a human triages it, and either runs it by hand or writes the operation. That loop is the feature. The escape hatch is not.

---

## 12. Decomposition patterns

| Shell shape | Pattern | Why |
|---|---|---|
| `a && b && c` (one intent) | **One operation** | Composition gets reviewed; failure path is code |
| `a && b && c` (unrelated) | **Separate operations** | Different effects, different risks |
| `if X then a else b` | **One operation**, condition inside | The agent shouldn't be the branch point for a mutation |
| `for f in *; do a $f; done` | **One operation** taking the collection | Bounded, one approval, one effect declaration |
| `a \| b \| c` (transform) | **One operation**, return structured data | The pipeline is an implementation detail |
| `a; check output; decide` | **Read op + judgment + write op** | This is legitimate agent work — reads are cheap, the write is gated |
| `watch a` | **One op + agent polling** | Never hold a connection open for observation |
| `a &` (background) | **One op returning `run_id`** | Architecture doc §13.5 |

The recurring principle: **the agent may compose reads freely; it must never compose mutations.** A sequence of mutations is a single intent that needs a single review — because the risk lives in the sequence, and a reviewer approving fragments never sees it.

---

## 13. The genuinely hard cases

Honest answers, including where the answer is "don't."

### 13.1 Interactive programs — `vim`, `top`, `psql`, `ssh`

**Don't.** There is no safe way to expose an interactive program: every editor and pager shells out (`:!cmd`), and a PTY is a terminal-injection surface. Model the intent instead:

```
vim config.yaml     →  config.set(key: enum, value)   with the file in the manifest
top                 →  metrics.snapshot()             structured
psql                →  db.query_named(query: enum, params)   §13.3
ssh other-host      →  a target of its own, or not at all
```

### 13.2 Editing files

`file.write(path, content)` is arbitrary execution one step removed (§10). Instead, one operation per *kind* of edit, with the file in the manifest:

```yaml
id: config.set
params:
  key:   {type: enum, values: [log_level, feature.new_checkout, worker.concurrency]}
  value: {type: string, pattern: '^[A-Za-z0-9._-]{1,64}$'}
effect:
  writes: ["/srv/workspace/api/config/dev.yaml"]
```

The implementation parses YAML, sets one known key, and writes it back. It cannot write a new key, cannot restructure the file, and cannot touch any other file. If you need to edit arbitrary source, that is what the developer's editor is for — not the agent's approval queue.

### 13.3 Database queries

`db.query(sql)` is both SQL injection and unbounded data access. Two workable shapes:

**Named queries** — best. The SQL lives in reviewed files.

```yaml
id: db.query_named
params:
  query:  {type: enum, values: [recent_orders, user_by_email, failed_jobs]}
  params: {type: object, schema_ref: "queries/{query}.params.json"}
returns: {rows: {type: array, max_items: 500}, truncated: {type: boolean}}
```

**A constrained builder** — acceptable when exploration is genuinely needed: enum table, enum columns, typed filters, mandatory limit. No raw SQL, no joins the agent composes, no subqueries.

Either way: read-only DB credentials for read operations, row caps, and column-level redaction for anything sensitive.

### 13.4 Genuinely novel, one-off work

Debugging something nobody anticipated is the real cost of this architecture. Don't pretend otherwise, and don't solve it with a hatch:

1. **Rich read-only coverage.** Most exploration is reading. Twenty good read operations cover most investigations, and they need no approval.
2. **The request valve.** The agent files a structured "I wanted to do X, here's why" record. A human triages: run it by hand, or write the operation. The agent's need is visible; its reach is not extended.
3. **Break-glass.** A human SSHes in directly, with session recording, logged to the same off-VM audit sink, with a written post-hoc review. Expect this; make it observable rather than pretending it won't happen.

The `OPERATION_UNAVAILABLE` rate tells you if this is working. Persistently high means write more operations. Zero probably means the agent has stopped trying — also worth investigating.

### 13.5 Long-running with progress

One operation, `run_id` returned immediately, output paged, cancel path available (architecture doc §13.5). Don't stream through the approval path, and don't hold a connection open.

### 13.6 Things needing privilege

```
❌  sudo.run(cmd)
❌  service.restart(unit: str)     with a writable unit directory (§10)
✅  service.restart(unit: enum[api, worker, scheduler])
```

Back it with the narrowest possible escalation — one polkit rule or one exact `sudoers` line with an absolute path and no wildcards:

```
devd ALL=(root) NOPASSWD: /usr/bin/systemctl restart api.service, \
                          /usr/bin/systemctl restart worker.service
```

Enumerate the units. Never `systemctl restart *` — a wildcard plus a writable unit directory is root.

---

## 14. Anti-pattern catalogue

Each one has shipped somewhere, wearing a reasonable justification.

| Anti-pattern | Why it fails | Instead |
|---|---|---|
| `run(cmd)` / `exec(cmd)` | It's a shell | Named operations |
| `git.raw(args: list[str])` | A shell with a prefix; `-c core.pager=sh` | One operation per git intent |
| `npm.run(script)` | Indirection through a writable file (§10) | `test.run(suite: enum)` |
| `make.target(name)` | Same | Named operations per target |
| `file.write(path, content)` | Arbitrary execution one step removed | `config.set(key: enum, value)` |
| `file.read(path)` | Reads `~/.aws/credentials` | `logs.tail(stream: enum)` |
| `db.query(sql)` | Injection + unbounded access | Named queries |
| `service.restart(unit: str)` | Free-string unit + writable unit dir | Enum units |
| `docker.run(image, args)` | Arbitrary container = arbitrary root | One operation per container job |
| `http.request(url)` | SSRF with your network position | Named, configured endpoints |
| Flags as a parameter | Effect declaration can no longer be true | Separate operations, or `mode: enum` |
| Path as a parameter | Traversal, symlinks, secrets | Enum key resolved server-side |
| Timeout as a parameter | DoS knob | Manifest |
| `mode: raw` fallback inside a good operation | The hatch, hidden in a good neighbourhood | Delete it |
| Operation that shells out internally | Injection moved, not removed | Library or argv list |
| `READ_ONLY` that writes a cache | Skips approval on a mutation | Classify honestly |
| Effect declaring `/srv/workspace` for one repo | Sandbox wider than needed | `/srv/workspace/{repo}` |
| `network: ["0.0.0.0/0"]` | Exfiltration channel | Named egress policy |
| Returning raw stdout by default | Context burn + parsing + injection | Structured `returns` |
| Uncapped array in `returns` | 40,000 filenames in the context window | `max_items` + `truncated` |
| One operation with an `action: enum` covering three effects | Declaration becomes a union | Three operations |

---

## 15. From runbook to operation set

### 15.1 Don't start from the runbook

Runbooks are aspirational and stale. Start from evidence of what people and agents actually do:

```bash
# 1. What do humans actually run on this box?
sort ~/.bash_history | uniq -c | sort -rn | head -50

# 2. What does CI run? (already curated, already reviewed)
grep -rhE '^\s+(run|script):' .github/workflows/ .gitlab-ci.yml

# 3. What does the agent ask for that you don't have?
#    — the OPERATION_UNAVAILABLE log, once you're running read-only (phase 3)
```

Source 3 is the best one, which is why the architecture doc's build order ships a read-only server first. A week of that log is worth more than a month of speculation.

### 15.2 Rank and batch

Score each candidate by **frequency × safety**, and do the safe and frequent ones first:

```
1. READ_ONLY, high frequency      →  ship first; no approval, immediate value
2. MUTATING_LOCAL, high frequency →  the core of the set
3. SERVICE_CONTROL                →  needs the privileged helper
4. NETWORK_EXEC                   →  needs the container sandbox
5. DESTRUCTIVE                    →  last; four-eyes, preconditions, snapshots
```

Batch by domain — all of `git.*`, then all of `test.*` — so you build one mental model at a time and the reviews stay coherent.

### 15.3 Expect the 80/20

In practice, **10–20 operations cover roughly 80% of real work.** A typical starting set:

```
READ_ONLY (7)        git.status · git.log · logs.tail · service.status
                     health.check · deps.outdated · disk.usage
MUTATING_LOCAL (5)   git.checkout · git.pull · test.run · lint.run · format.run
SERVICE_CONTROL (2)  service.restart · service.reload
NETWORK_EXEC (3)     deps.install · build.container · api.refresh_dev
DESTRUCTIVE (2)      deps.reinstall · cache.clear
```

If you are past 60 operations, look for a missed abstraction — you have probably converted commands rather than intents, and several of those operations are the same intent wearing different flags.

### 15.4 Measure

| Metric | Healthy | Unhealthy |
|---|---|---|
| `OPERATION_UNAVAILABLE` / week | Low and falling | Rising — write operations, don't loosen |
| Operations never used in 90 days | Few | Many — delete them; unused surface is pure risk |
| Approvals per useful outcome | Low | High — altitude too low, split intents wrongly |
| `SANDBOX_VIOLATION` | ~0 | Any — a manifest is wrong; fix the bug, don't widen |
| Approval latency | Seconds to minutes | Near-zero — people are clicking through |

Two of those are about people rather than code, and they're the ones that predict failure earliest.

---

## 16. Templates

### 16.1 Manifest

```yaml
# devd/operations/<domain>_<verb>.yaml
id: domain.verb
version: 1                       # bump on ANY behavioural change — it is inside the hash
summary: "One sentence, present tense, with an object."

params:
  thing: {type: enum, values: [a, b, c]}
  name:  {type: string, pattern: '^[A-Za-z0-9._-]{1,64}$', max_length: 64,
          no_leading_dash: true}
  count: {type: integer, min: 1, max: 100, default: 10}

effect:
  description: |
    What this does, in a sentence or two a reviewer can evaluate.
    What it explicitly does NOT do, where someone might assume otherwise.
  reads:    ["/path/{thing}"]
  writes:   []
  network:  []
  services: []

risk: READ_ONLY                  # READ_ONLY | MUTATING_LOCAL | SERVICE_CONTROL
                                 # NETWORK_EXEC | DESTRUCTIVE
approval: not_required           # not_required (READ_ONLY only) | required | four_eyes

preconditions: []                # mandatory for DESTRUCTIVE

reversible:
  kind: none                     # none | git_ref | snapshot_required | scripted
  note: "Say plainly what cannot be undone."

returns:
  field: {type: string}
  items: {type: array, items: string, max_items: 50}
  truncated: {type: boolean}

sandbox:
  mode: namespace                # none | namespace | container
  timeout_s: 60
  max_output_bytes: 262144
```

### 16.2 Implementation

```python
# devd/operations/domain_verb.py
from devd import operation, Precondition, Result

@operation("domain.verb", version=1)
async def domain_verb(ctx, thing: str, name: str, count: int = 10) -> Result:
    """One sentence — must match the manifest summary."""
    target = ctx.resource(thing)                       # config lookup, never a raw path

    ctx.require(Precondition("thing.state", await target.state()))

    out = await target.run(["binary", "subcommand", "--flag", name])   # argv list

    return Result(
        summary=f"Human-readable one-liner about {name}",
        data={"field": parse(out), "items": items[:50], "truncated": len(items) > 50},
    )
```

### 16.3 Test

```python
# tests/operations/test_domain_verb.py

async def test_happy_path(vm, op):
    r = await op.run("domain.verb", thing="a", name="ok")
    assert r.data["field"] == "expected"

@pytest.mark.parametrize("bad", [
    "a; rm -rf /", "a && curl x|sh", "$(id)", "`id`",
    "-c core.pager=sh", "--exec=sh",                 # option injection
    "a\x00b", "a\nb", "a\rb",                        # control characters
    "../../../etc/passwd", "а",                      # traversal, Cyrillic homoglyph
])
async def test_rejects_hostile_params(op, bad):
    with pytest.raises(InvalidParameter):
        await op.run("domain.verb", thing="a", name=bad)

async def test_effect_conformance(vm, op, syscall_audit):
    """The manifest is a promise. This is the test that it is kept."""
    with syscall_audit() as audit:
        await op.run("domain.verb", thing="a", name="ok")
    manifest = load_manifest("domain.verb")
    assert audit.paths_written <= expand(manifest.effect.writes, thing="a")
    assert audit.paths_read    <= expand(manifest.effect.reads,  thing="a")
    assert audit.connections   <= expand(manifest.effect.network)

async def test_precondition_drift(vm, op):
    approved = await op.propose("domain.verb", thing="a", name="ok")
    await vm.mutate_state("a")                        # change it underneath
    with pytest.raises(PreconditionDrift):
        await op.execute(approved)

async def test_returns_are_capped(vm, op):
    await vm.create_many_items("a", 10_000)
    r = await op.run("domain.verb", thing="a", name="ok")
    assert len(r.data["items"]) <= 50 and r.data["truncated"] is True
```

The conformance test is the one that matters most and the one most likely to be skipped. **It is what converts the effect declaration from documentation into a guarantee** — without it, §5.2 of the architecture doc is prose.

---

## 17. Review checklist

Print this. An operation is the perimeter; review it like production code, because it is.

```
ALTITUDE
  □ One true, specific sentence describes it
  □ A reviewer could approve it from the prompt alone, with no external knowledge
  □ It is one intent, not a menu of intents behind an `action` param
  □ The name is a domain and a verb, with no "raw", "generic", "exec", "custom"

PARAMETERS
  □ Every parameter is fixed, enum, or a bounded validated scalar
  □ No path, URL, flag, command, script name, timeout, or sandbox setting
  □ Patterns are anchored fullmatch with max_length
  □ no_leading_dash set on every string parameter (or justified in the diff)
  □ Control characters rejected
  □ No parameter selects behaviour from a file the agent can write          ← §10
  □ Each parameter fills a WHOLE argv element

EFFECT
  □ writes / reads are the narrowest paths that work
  □ network defaults to [] and lists exact hosts otherwise; never 0.0.0.0/0
  □ services enumerated
  □ Description says what it does AND what it notably does not do
  □ Verified against an actual strace, not from memory                      ← §5
  □ Declaration is true for EVERY parameter combination, not just the common one

PRECONDITIONS
  □ Present for DESTRUCTIVE (mandatory)
  □ Checked adjacent to the effect, not a round trip away
  □ An atomic form is used where one exists
  □ Human-readable in the approval prompt

RISK
  □ Class is honest — no "READ_ONLY" that writes a cache
  □ NETWORK_EXEC if it downloads and runs anything                          ← §7.1
  □ Container sandbox for NETWORK_EXEC
  □ four_eyes for DESTRUCTIVE
  □ reversible is honest; kind: none prompts "should this snapshot first?"

IMPLEMENTATION
  □ No shell=True, os.system, or string-built commands
  □ Library preferred over subprocess
  □ Environment constructed, not inherited
  □ Errors are typed; raw stderr is never relayed to the agent
  □ Idempotent where the underlying operation allows

RETURNS
  □ Structured, typed, declared in the manifest
  □ Every array has max_items and a truncated flag
  □ Raw output is paged and bounded, not the default return
  □ No secrets in the return path

TESTS
  □ Happy path
  □ Hostile parameters (the parametrized list, including leading dashes)
  □ Effect conformance under syscall audit                                  ← the important one
  □ Precondition drift
  □ Return capping
  □ Timeout behaviour

MANIFEST HYGIENE
  □ version bumped if ANY behaviour changed                                 ← §18
  □ Manifest and implementation summaries match
  □ Manifest schema-validates in CI
```

---

## 18. Lifecycle

### 18.1 Changing an operation

**Bump `version` on any behavioural change.** The version and manifest hash are inside the approval hash (architecture doc §5.6), so a bump invalidates in-flight approvals — which is exactly right when behaviour changed, because the human approved the old behaviour.

```
Behaviour changed?           → bump version. Always.
Only the summary wording?    → bump anyway; it's what the human reads.
Only a code comment?         → no bump.
Effect declaration changed?  → bump, and flag it for security review.
```

Enforce with a CI check that hashes the manifest **and** the implementation together and fails if the pair changed without a version bump. Manual discipline fails here within about three months.

### 18.2 Deprecating

```yaml
deprecated:
  since: "2026-09-01"
  reason: "Replaced by api.refresh_dev, which includes the health check."
  replacement: api.refresh_dev
  removal_after: "2026-12-01"
```

The manifest keeps serving it; `list_operations` marks it deprecated so the agent prefers the replacement; usage is logged. Remove it after the window.

**Delete unused operations.** An operation nobody has used in 90 days is surface with no benefit. Check the usage metric before every release.

### 18.3 Adding one

Keep this cheap or §10's escape hatch wins on convenience alone:

```
1. Copy the templates (§16)
2. strace the command to get the effect declaration right
3. Fill in the manifest, run the schema check
4. Implement; run the test suite
5. PR → security review for anything above READ_ONLY;
        one reviewer for READ_ONLY
6. Deploy devd; manifest hash changes; in-flight approvals invalidate (correct)
```

**Target: an experienced author ships a READ_ONLY operation in under an hour.** If it takes a day, someone will build the escape hatch instead, and they will be right that it's faster. Invest in the template, the test harness, and the fast review path — that investment *is* a security control.

---

## 19. When not to make it an operation

Not everything belongs here. Saying no keeps the set small and reviewable.

| Situation | Do this instead |
|---|---|
| One-off investigation, never to recur | Break-glass: a human SSHes in, recorded and audited |
| It belongs in CI | Put it in CI. It gets review, logs, and artifacts for free |
| A human should decide, not an agent | Leave it manual. Not everything should be automatable |
| Needs true privilege (user management, kernel, firewall) | Not through this server. Separate tooling, separate authorization |
| Touches production | Not from a dev-VM server. Different system, different review |
| Used once a year | A documented runbook is fine. An unused operation is pure risk |
| You can't write a true, specific effect description | That's a signal the intent isn't clear yet — go back to §1 |

That last row is the general test. **If you cannot describe the effect truthfully and specifically, you do not understand the operation well enough to let an agent trigger it** — and no amount of sandboxing fixes that.

---

## 20. Summary

```
  A SHELL COMMAND                          AN OPERATION
  ───────────────                          ────────────
  npm run test:unit             ──▶        test.run(repo: enum, suite: enum)

  effect: unknowable                       effect: declared, enforced by the sandbox
  params: an arbitrary string              params: two closed sets
  approval: "read this and hope"           approval: "runs the unit suite in api,
                                                     writes only cache and reports,
                                                     no network"
  returns: text to parse                   returns: {passed, failed, failures[]}
  review: impossible                       review: a 40-line YAML + Python diff
  blast radius: the account                blast radius: two directories
```

**The conversion in one line:** find the intent, close the parameter sets, declare the effect, and let the sandbox hold you to the declaration.

Three things to keep checking as the set grows:

1. **Altitude** (§1) — the source of almost every bad operation.
2. **The indirection trap** (§10) — the source of almost every operation that *looks* good and isn't.
3. **Coverage pressure** (§15.4) — the thing that, if you ignore it, eventually produces `POST /ops/exec` and undoes all of this.
