---
name: atomic-commits
description: Review a messy Git working tree, preview intent-based groups, and create small Conventional Commits safely. Use when the user asks to split many pending changes into commits, organize Git history by feature/fix/refactor, review a commit plan, or dry-run a large set of changes. Git execution is restricted to git status, git diff, git add, and git commit -m.
---

# Atomic Commits

Turn a crowded working tree into a readable history of focused commits. **Infer intent from actual changes, not filenames.** Preserve user work and favor an uncommitted change over an incorrect commit.

## Hard execution boundary

Only the following Git command families may be executed:

- `git status` (e.g. `git status --short`, `git status --porcelain=v1`)
- `git diff` (e.g. `git diff`, `git diff --cached`, `git diff --stat`, `git diff --check`, `git diff --no-index` is **not** allowed because it can read arbitrary non-repository files; use normal repository diffs only)
- `git add` (explicit paths, or `git add -p` for independently stageable hunks)
- `git commit -m "type: description"` (exactly the `-m` option, one message)

**Do not execute any other Git command**: no `reset`, `restore`, `stash`, `clean`, `show`, `log`, `push`, `pull`, `fetch`, `switch`, `checkout`, `branch`, `rebase`, `rm`, `ls-files`, `rev-parse`, or `apply`. No chained commands, shell scripts, pipelines, wrappers, aliases, secondary Git tools, or subprocesses that bypass this allowlist. Do not run tests, build commands, formatters, or source-modifying tools. Read-only source inspection through provided file-reading facilities is permitted; do not invoke arbitrary shell commands for it. Git hooks triggered by `git commit` may execute automatically; **do not intentionally bypass them** or claim that the four-command restriction also limits hook internals.

Never use `git add .`, `git add -A`, `git add --all`, glob/wildcard staging, `git commit -a`, `--amend`, `--no-verify`, or Git configuration changes. For `git add`, supply named repository-relative paths after `--` to disambiguate flags. Interactive patch staging may use `git add -p -- path/to/file` **only if reliable human/agent interaction and review of each hunk is available**.

Do not edit any code, config, or data file. Do not remove or revert user changes. No push is permitted.

## Modes

Infer mode from the user's request; do not introduce a confirmation prompt just because a safe preview can be produced.

- **Execute (default):** inspect, present a concise **commit plan preview**, then stage, verify, and commit each safe group autonomously. Pause only at unsafe/uncertain operations.
- **Dry run / preview only:** inspect and present the commit plan, risk flags, dependencies, and leftovers, **without invoking `git add` or `git commit`**. Make it clear that no commit was created.
- **Restricted scope:** if the user names specific features/paths, touch only that scope; still inspect the complete staged index before any commit to prevent accidental inclusion of out-of-scope work.

A preview is a plain-text table/list in the agent response, **not** an interactive GUI. Do not promise confidence scores with spurious precision.

## Commit quality

A commit captures **one independently understandable change**, potentially spanning UI, API, migrations, tests, and documentation. Group by behavior and reason for change, **not directory, file type, or an arbitrary maximum number of files**.

- Combine necessary cross-layer edits for the same feature.
- Include supporting tests/docs with their implementation when they serve only that implementation.
- Separate unrelated fixes, formatting, refactors, and new behavior where safely possible.
- Sequence commits so prerequisites precede dependents; when an independent split would make intermediate commits misleading or obviously incomplete, treat the tightly coupled change as one unit.
- Favor an honest larger cohesive commit over artificial micro-commits, but never combine unrelated work merely to finish everything.

### Mixed-file detection

A file can contain several features or a feature plus unrelated formatting. Inspect its diff at hunk level and label it **mixed** if distinct intents appear. Before planning:

1. Check whether distinct changes are in separate, independently stageable hunks.
2. Use `git add -p` only with reliable control of selections; verify the entire staged diff after selection.
3. If unrelated edits share one inseparable hunk, **do not pretend partial staging works**. Leave that file (and any group requiring it) uncommitted unless all its changes genuinely form one coherent unit.
4. If a file is partially staged, remember the remaining working-tree changes can later be staged only after verifying the previous commit. Never accidentally re-stage already committed intent.
5. Renames, deletions, new files, submodules, binary changes, and generated files require extra caution because patch separation or inspection may not be possible with permitted tools. Leave them uncommitted if content and intent cannot be verified.

## Conventional Commit messages

Use **English**, lowercase except proper nouns, format:

`type: short imperative description`

Accepted types: `feat`, `fix`, `refactor`, `perf`, `test`, `docs`, `style`, `chore`, `build`, `ci`.

Examples:

- `feat: add process monitoring screen`
- `fix: prevent duplicate webhook notifications`
- `refactor: extract external api rate limiter`
- `perf: reduce conversation filtering latency`
- `chore: configure sql server container`

Prefer specific descriptions of roughly 3–8 words, without trailing periods. Avoid vague messages (`fix: stuff`, `chore: misc`) and file-name inventories. Do not use fabricated ticket IDs or scopes.

## Procedure

### 1. Read the index and worktree first

Run `git status --short`, `git diff --stat`, `git diff`, `git diff --cached --stat`, and `git diff --cached` (scope large diffs to specific paths as needed). Review untracked-file contents using read-only file inspection when available; untracked content **does not appear** in ordinary `git diff`. `git status` alone never establishes the safety of an untracked file.

**If the index is already staged at entry, do not stage or commit anything.** Show the pre-existing staged changes, explain that the four-command allowlist cannot safely unstage them, and request manual resolution or fresh instructions after the index is clean. The staged index belongs to the user, even when it looks coherent.

Flag risks: `.env` and credential files, tokens/private keys, local settings, caches, generated build outputs, large unknown/binary files, potential secrets embedded in ordinary code, and unclear deletions. Do not stage suspected secrets, even when they sit beside legitimate changes in a file; if inseparable, leave the file alone.

### 2. Analyze intent and dependencies

Read relevant diffs before grouping. Map each proposed group to:

- An intent (`feat`, `fix`, `refactor`, etc.)
- Exactly which **paths and, if needed, hunks** belong to it
- Dependencies on other proposed groups
- Potential risk or uncertainty
- Leftover changes that cannot safely be committed

Avoid treating a shared file as belonging wholly to one group if it contains unrelated changes. For new files, only include files whose contents you can actually inspect. Read diffs until the classification is evidence-based, not just a guess from naming.

### 3. Display commit preview

Before writing commits, show a compact plan like:

```text
Proposed commits (example)
1. feat: add process monitoring screen — 6 files
2. fix: prevent duplicate notifications — 3 files, 1 partial hunk
3. refactor: extract rate limiting handler — 4 files

Excluded: .env.local (sensitive); src/shared.ts (inseparable mixed changes)
Order: #3 before #2 only if #2 depends on #3
Mode: execute | dry-run
```

Make it clear that file counts and group names are proposals until verified. The preview is **not permission to add unsafe files**. In execute mode continue after preview without requiring approval for confidently safe groups; in dry-run mode stop here.

### 4. Stage one group and verify

Stage whole, fully reviewed files via `git add -- path1 path2`. Stage separate hunks only via controlled `git add -p -- path`. Never stage a directory if it can include unreviewed content.

Immediately run:

- `git diff --cached --stat`
- `git diff --cached`
- `git diff --cached --check`
- `git status --short`

Verify **every staged change** belongs to just the current group, contains no suspected secrets, and was not staged inadvertently. A whitespace warning requires inspection, not an automatic blanket refusal; if unclear, stop.

**Critical stop rule:** if staging does not match the plan, stop without committing. The allowed commands do not include a safe way to unstage a path/hunk. Never try a destructive workaround or run a forbidden command. Report precisely what is staged so the user can repair it manually.

### 5. Commit, then check again

Only after exact verification execute `git commit -m "type: description"`. Inspect `git status --short` and `git diff --cached` after each commit before proceeding. If a hook fails, the index remains staged, the index changes unexpectedly, or a command has an unexpected side effect, **stop**; do not retry by circumventing hooks or rewriting history.

If hooks modify files, treat them as fresh, unreviewed changes. Never assume success merely from the commit command being attempted. Continue group by group only while the staging index is understood and clean.

### 6. Summary and leftovers

Report committed messages, concise intent, and what remains modified/untracked with reasons. Clearly distinguish **planned**, **committed**, **skipped**, and **blocked**. Mention that tests/builds were not run because this skill permits only Git commands. Never claim changes were pushed or validated by a test suite.

## Safety rules at a glance

- Protect all user work; never clean, reset, restore, or rewrite it.
- A staged index that predates this skill is a hard stop.
- Preview first; dry run is read-only.
- Stage only inspected paths or controlled hunks; inspect the exact staged diff before every commit.
- No automatic staging of unknown files or possible secrets.
- When intent or atomic separation is uncertain, **leave it uncommitted** and explain why.
- Remain within the four authorized Git command families, including on failures.

## Invocation examples

- `Use $atomic-commits to group my pending changes into small commits by functionality.`
- `Use $atomic-commits in dry-run mode. Show the proposed commits, but do not stage or commit.`
- `Use $atomic-commits only for the notification feature; leave everything else untouched.`
