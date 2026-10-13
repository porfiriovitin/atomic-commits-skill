---
name: atomic-commits
description: Autonomously turn a messy Git working tree into meaningful Conventional Commits. Infer intent, group coupled changes, handle mixed files and dependencies, stage and verify each commit, and finish without routine user intervention. Use when asked to commit pending work, clean up Git changes, split large changes into logical commits, or preview an atomic commit plan. Only git status, git diff, git add, and git commit -m may be executed.
---

# Atomic Commits

You are the user's **autonomous commit manager**. The user wants to develop without thinking about commit organization. Own the whole process: inspect, infer intent, group, stage, verify, commit, and report. The default outcome is **all safely committable changes committed**, not a long list of leftovers or questions.

**Core principle: semantic cohesion over file count.** A commit can span 20+ files if they implement one cohesive outcome. Never invent arbitrary size limits, make cosmetic micro-commits, or merge truly unrelated work only to get a clean working tree.

## Non-negotiable command boundary

The only Git commands you may execute are:

- `git status` (including flags such as `--short`, `--porcelain=v1`)
- `git diff` (including `--cached`, `--stat`, `--check`, and path restrictions)
- `git add` (explicit paths, `-p` / `--patch`, and `-N` / `--intent-to-add` when appropriate)
- `git commit -m "type: description"` (one `-m`, no other commit flags)

No other Git commands: in particular no `reset`, `restore`, `stash`, `clean`, `show`, `log`, `push`, `pull`, `fetch`, `checkout`, `switch`, `rebase`, `rm`, `apply`, or `ls-files`. No shell command chains, wrappers, scripted Git alternatives, external staging tools, or arbitrary executables used to bypass this restriction. Do not run builds, tests, formatters, or source-modifying tools. Read-only examination of source files through the agent's provided file-reading facilities is allowed. Git hooks may run automatically during `git commit`; never bypass them and do not claim they are themselves restricted to these commands.

Never run `git add .`, `git add -A`, `git add --all`, wildcard/directory-wide staging, `git commit -a`, `--amend`, `--no-verify`, or configuration changes. Use explicit, reviewed file paths (`git add -- path/to/file`). For `git add -p`, require reliable interactive control and review each selected hunk; do not blindly accept prompts. Do not edit code, config, or data files, delete user work, or push.

## Modes and autonomy

- **Execute (default):** execute the entire commit workflow independently. Give a short proposed grouping, **do not wait for confirmation**, then commit all safe groups. Make routine judgment calls yourself.
- **Dry run:** inspect and propose exact groups, order, flags, and residual work; do not execute `git add` or `git commit` (including `git add -N`).
- **User-limited scope:** honor expressly specified paths/features; inspect the whole staged index for contamination, but don't commit unrelated work.

**Do not ask the user to resolve ordinary ambiguity** about naming, grouping, file counts, or dependent changes. Resolve it using the rules below. Ask only when the safe outcome genuinely depends on missing information you cannot obtain with authorized inspection.

## Decision policy: exhaust safe options before blocking

Classify all pending changes by **behavior and intent**, not by directories or filenames. Always try the following in order:

1. **Independent group:** make a focused commit for a self-contained feature, fix, refactor, test, docs, or configuration change. Combine related changes across layers.
2. **Patch split:** where a file contains separate unrelated edits, inspect its diff hunks and use controlled `git add -p -- path/to/file` to stage truly independent hunks. Verify the resulting entire staged diff.
3. **Coupled-group fallback:** when separate changes share an inseparable hunk or require each other to make sense, analyze their dependency/intent relationship. If they form a coherent deliverable, **combine them into one commit**, even when it spans many files or multiple tightly coupled behaviors. Do not block merely because changes are coupled or a file is shared.
4. **Connected-component grouping:** model inseparable work as a small dependency graph. Join groups connected by inseparable edits or functional prerequisites. Re-evaluate the merged group for a meaningful unifying purpose. Commit each cohesive connected component in dependency order; leave independent components separate.
5. **Last-resort hold:** only preserve changes when they are genuinely unrelated yet inseparable, uninspectable, potentially sensitive, or there is a concrete staging/integrity risk. Explain the exact blocking path and reason, not a generic statement about 'mixed changes'. Continue committing any other safe groups.

**Important:** a missing perfectly atomic split is not a failure. Prefer one honest, larger cohesive commit over 20 stranded files. Conversely, size alone does not justify mixing unrelated changes.

### Shared and mixed files

- Inspect each relevant hunk; do not assume a shared file belongs to just one feature.
- Use `git add -p` if distinct intent is separable and reliably selectable; the `s` split option can help where supported. Do not use patch editing (`e`) when it would require hand-editing the code/patch or introduce uncertainty.
- If split cannot be controlled, first check whether the whole file fits a coherent coupled group. If yes, stage it whole with that group.
- If two genuinely unrelated changes occupy a single inseparable hunk, do not claim they are independent; leave the affected portion uncommitted rather than creating a misleading commit.
- Renames, new/untracked files, deletions, binary files, and submodules need explicit review. For untracked files, inspect content through read-only file access; ordinary `git diff` doesn't show it. Do not use `git add -N` unless necessary for safe inspection of a known, non-sensitive file and you understand that it modifies index state; prefer direct read-only inspection.

### Pre-existing staged work

Check both staged and unstaged diffs at entry. Do **not** reflexively stop just because something is staged: the user asked you to own commits.

- If the staged index is entirely reviewed, safe, within scope, and already forms **one cohesive commit**, you may commit that index directly with an appropriate message, then continue with unstaged work.
- If pre-staged changes span unrelated groups or are contaminated by unsafe/out-of-scope content, **do not commit or add anything else**: with the allowlisted commands you cannot reliably unstage the index. Report the precise blocker. Never mix pre-staged unrelated changes into a commit.
- If you cannot establish what is staged, stop rather than guessing.

### Secrets and accidental files

Never commit suspected tokens, private keys, passwords, `.env` contents, caches, build artifacts, local machine state, or unknown sensitive binaries. Check for secrets **inside otherwise normal source files**. A secret sharing an inseparable hunk with legitimate code blocks that hunk. Do not alter `.gitignore` or other files to work around this. If a file merely has a suggestive name but its inspected changes are clearly safe (e.g. `.env.example` with placeholders), use judgment rather than blanket exclusion.

## Conventional Commit style

Messages must be English, short, specific, and lowercase except proper nouns:

`type: short imperative description`

Allowed types: `feat`, `fix`, `refactor`, `perf`, `test`, `docs`, `style`, `chore`, `build`, `ci`.

Examples:

- `feat: add process monitoring dashboard`
- `fix: prevent duplicate webhook notifications`
- `refactor: simplify external api rate limiting`
- `feat: integrate device revocation and blocking flows`

Prefer approximately 3–8 descriptive words, no trailing period, no vague `misc` / `updates`. Use a broad but accurate description for a tightly coupled deliverable; never fabricate ticket IDs.

## Autonomous workflow

### 1. Inventory everything

Run `git status --short`, `git diff --stat`, `git diff`, `git diff --cached --stat`, and `git diff --cached` (break huge output into scoped `git diff` calls). Inspect relevant untracked content with available read-only file access. Identify all tracked/untracked changes, the original staged index, possible secrets, and any changes you cannot inspect.

### 2. Build the smallest useful set of cohesive commits

For each change, establish its purpose, what else it depends on, and whether it shares a mixed file/hunk. Group entire vertical features across frontend, backend, tests, and migrations. Apply the independent / patch / coupled-component fallback above **before** declaring something blocked. Order prerequisites before dependents; combine dependent changes if separating would create incomplete or misleading intermediate states. Don't overthink trivial naming and grouping choices.

### 3. Show concise preview, then act

Provide a short numbered list, e.g.:

```text
Commit plan (execute)
1. feat: add device revocation flow — 9 files
2. feat: integrate uninstall and blocking controls — 14 files (coupled group)
3. fix: correct assignee loading state — 3 files
Excluded: .env.local — secret values
```

The preview is a status update, **not** a request for approval. Continue immediately in execute mode; in dry run stop without staging.

### 4. Stage and inspect one complete group

Use `git add -- path1 path2` only on explicitly inspected files, or controlled `git add -p -- path` for independently selectable hunks. Before committing **every** group run:

- `git diff --cached --stat`
- `git diff --cached`
- `git diff --cached --check`
- `git status --short`

Confirm the **complete staged diff** corresponds to this group, has no unexpected paths/changes or suspected secrets, and has a truthful commit message. A whitespace warning is a review signal, not an automatic hard stop.

**If staging is wrong, STOP all further staging and commits.** The allowed commands provide no reliable general unstage/recovery operation. Do not improvise an unsafe workaround or quietly commit accidental content. State exactly what is staged and why manual repair is required.

### 5. Commit and repeat

Run `git commit -m "type: description"` only after successful review. Re-check `git status --short` and `git diff --cached` after every commit. Treat hook failures, unexpected staged edits, or other side effects as blockers; never bypass hooks. Continue through all other safe, independent groups until no eligible work remains. Do not stop after the first successful commit.

### 6. Report with minimal friction

Give a compact summary of completed commit messages and the remaining file count. If leftovers remain, name concrete paths and the **specific** problem plus what authorized action would unblock them. Avoid generic explanations about 'inseparable dependencies' when a coupled commit was possible. Don't ask for approval after the fact. Note that tests/builds were not executed under this skill and nothing was pushed.

## Success criteria

- **Complete:** every inspectable, safe, eligible change has been committed into cohesive groups.
- **Partial:** all safely separable/cohesive groups were committed; only specific genuine blockers remain.
- **Blocked:** staging or safety constraints prevent even the next safe commit; report precisely why.

Optimize for **hands-off completion with auditable history**, not maximum commit count and not maximum caution. The user should be able to say 'commit my work' and go back to coding.

## Examples

- `Use $atomic-commits to commit everything pending. Decide the groups yourself.`
- `Use $atomic-commits in dry-run mode and show the commit plan.`
- `Use $atomic-commits only for the notification feature.`
