<table>
  <tr>
    <td width="220" valign="middle">
      <img src="https://octodex.github.com/images/dodgetocat_v2.png" alt="Dodgetocat from GitHub Octodex" width="200" style="border-radius: 16px;">
    </td>
    <td valign="middle">
      <h1>Atomic Commits</h1>
      <p>A Codex skill that turns a crowded Git working tree into small, focused Conventional Commits.</p>
      <p>It reads the actual changes, groups them by intent, previews a commit plan, and commits each safe group in dependency order.</p>
      <p>Changes are grouped by behavior and purpose, rather than directory or file type. A single feature can include UI, API, tests, and documentation in one coherent commit.</p>
    </td>
  </tr>
</table>

## Installation

### Install from GitHub with `$skill-installer`

Ask Codex to install the skill directly from its GitHub repository:

```text
Use $skill-installer to install the skill from https://github.com/<owner>/<repo>/tree/main/atomic-commits.
```

Replace `<owner>` and `<repo>` with the GitHub repository owner and name. If the repository uses a different branch, replace `main` as well. The URL should point to the `atomic-commits` directory containing `SKILL.md`.

The installer places the skill in `$CODEX_HOME/skills/atomic-commits` (by default, `~/.codex/skills/atomic-commits`). After installation, the skill is available on your next turn.

### Manual installation

Copy the `atomic-commits` directory into your Codex skills directory:

```text
~/.codex/skills/atomic-commits/SKILL.md
```

On Windows, the default location is:

```text
%USERPROFILE%\.codex\skills\atomic-commits\SKILL.md
```

The skill instructions are in [atomic-commits/SKILL.md](atomic-commits/SKILL.md).

## Usage

Open the Git repository you want to organize in Codex, then invoke the skill:

```text
Use $atomic-commits to group my pending changes into small commits by functionality.
```

### Preview without committing

```text
Use $atomic-commits in dry-run mode. Show the proposed commits, but do not stage or commit.
```

### Limit the scope

```text
Use $atomic-commits only for the notification feature; leave everything else untouched.
```

## Modes

| Mode | Behavior |
| --- | --- |
| Execute (default) | Inspect changes, show a plan, then stage, verify, and commit safe groups. |
| Dry run | Show proposed commits, dependencies, risks, and leftovers without staging or committing. |
| Restricted scope | Process only the requested features or paths while checking the full staged index. |

In execute mode, safe groups proceed after the preview without a separate approval prompt. Uncertain or unsafe changes are left uncommitted.

## Workflow

1. Inspect the working tree and staged index, including the contents of untracked files.
2. Identify each change's intent, dependencies, and any unrelated edits within the same file.
3. Preview proposed commit messages, affected files or hunks, and excluded changes.
4. Stage one group using explicit paths or carefully controlled patch selection.
5. Review the entire staged diff before committing.
6. Check the index after each commit and report completed commits and remaining changes.

Example preview:

```text
Proposed commits
1. feat: add process monitoring screen — 6 files
2. fix: prevent duplicate notifications — 3 files, 1 partial hunk
3. refactor: extract rate limiting handler — 4 files

Excluded: .env.local (sensitive); src/shared.ts (inseparable mixed changes)
Order: #3 before #2 if the notification fix depends on the refactor
Mode: execute
```

## Commit messages

Messages use English and the following format:

```text
type: short imperative description
```

Supported types: `feat`, `fix`, `refactor`, `perf`, `test`, `docs`, `style`, `chore`, `build`, and `ci`.

Examples:

```text
feat: add process monitoring screen
fix: prevent duplicate webhook notifications
refactor: extract external api rate limiter
perf: reduce conversation filtering latency
```

## Safety and limitations

- **Start with an empty staging index.** If changes are already staged, the skill stops and asks for manual resolution before proceeding.
- Only `git status`, repository `git diff`, explicit `git add`, and `git commit -m` are permitted. `git diff --no-index` is excluded.
- No reset, restore, stash, clean, history rewriting, Git configuration changes, or push.
- No blanket staging with `git add .`, `git add -A`, or wildcards.
- Source files are inspected without editing them. Tests, builds, and formatters are not run by the skill; commit hooks may run automatically and are never intentionally bypassed.
- Suspected secrets, unverified files, and inseparable unrelated changes remain uncommitted.
- Partial staging is used only when each hunk can be reliably selected and reviewed.
- Unexpected staged changes or hook failures stop execution. The skill does not attempt to unstage files or bypass hooks.

The final report distinguishes committed, skipped, and blocked work, and explains any leftovers. A successful run organizes Git history; it does not establish that the code passes tests.
