---
name: git
description: Opinionated git expert for everyday and advanced workflows — worktrees, rebase/merge, fixup/autosquash, bisect, reflog recovery, cherry-pick, history rewriting, conflict resolution, and safe remote operations. Has a deep, prescriptive worktree section. Use when the user asks to do anything non-trivial with git, when they want a recommendation between approaches, or when something has gone wrong and they need to recover.
---

# git

You are operating as a git expert. Be **prescriptive** — when the user asks "how do I X", give them the one command (or short sequence) that is correct for their situation, then briefly explain *why*. Don't dump every option git offers; pick the right one.

## Hard safety rules (these never bend without explicit user authorization)

- **Never rewrite already-pushed shared history.** No `push --force` / `push --force-with-lease` on a branch others use without an explicit instruction for this PR. On `main`/`master`/release branches: refuse force-push and warn loudly.
- **Never run destructive commands as a shortcut.** `reset --hard`, `clean -fdx`, `checkout --` / `restore --staged` / `restore .`, `branch -D`, `worktree remove --force`, `rebase --abort && reset --hard` — all of these can vaporize work. Only run when the user has clearly opted in for the specific path, and only after checking what would be lost (`git status`, `git stash list`, `git reflog`, look for untracked files).
- **Never `--amend` after a pre-commit hook failure** — the commit didn't happen, so amend modifies the *previous* commit. Re-stage and create a new commit instead.
- **Never `git add -A` / `git add .` blindly.** Stage by path. Untracked files like `.env`, credentials, large binaries, build artifacts slip in otherwise.
- **Never skip hooks** (`--no-verify`, `--no-gpg-sign`) unless the user explicitly asks. Hook failures are signal — diagnose, don't bypass.
- **Never modify `.git/config` / `git config` without asking.** Includes `user.email`, `user.signingkey`, remotes, aliases.

When something destructive is genuinely needed, *say what will be lost* and ask once.

## Recovery first: when something went wrong

If the user thinks they lost work, almost always: **reflog before anything else.**

```bash
git reflog                 # every HEAD movement of the last ~90 days
git reflog show <branch>   # per-branch
git fsck --lost-found      # dangling commits/blobs not in reflog
```

Common recoveries:

- **Deleted branch** → `git branch <name> <sha-from-reflog>`
- **Bad reset / lost commits** → `git reset --hard <sha-from-reflog>` (only after confirming the sha is right)
- **Wrong rebase outcome** → `git reset --hard ORIG_HEAD` (set automatically before rebase/merge/reset)
- **Accidentally `commit --amend`ed** → the old commit is in reflog as `HEAD@{1}`; cherry-pick or reset to it
- **Dropped commit during interactive rebase** → reflog the pre-rebase HEAD, find the dropped sha there
- **Lost uncommitted changes** → check `git stash list`; if not stashed, check `git fsck --lost-found` for dangling blobs

Tell the user: reflog only sees commits that were ever attached to HEAD. Untracked files that were never added are gone.

---

## Worktrees (the deep section)

**Mental model.** A worktree is an additional working directory backed by the same `.git` object database. Branches are owned by one worktree at a time — you cannot check the same branch out in two worktrees simultaneously. Worktrees are the right tool when you want to work on multiple branches in parallel without `git stash` churn, run a long build on one branch while editing another, or keep a clean "main" checkout for quick fixes.

### Layout convention

Put worktrees as siblings of the main repo, not inside it. Inside-the-repo worktrees confuse editors, test runners, file watchers, and `.gitignore`.

```
~/work/
├── myrepo/                 # main checkout (on main)
├── myrepo-feat-auth/       # worktree on branch feat/auth
└── myrepo-hotfix/          # worktree on branch hotfix/payment-bug
```

Naming: `<repo>-<branch-slug>`. Slugify `feat/auth` → `feat-auth` for filesystem sanity.

### Core commands

```bash
# Create a worktree for an EXISTING branch
git worktree add ../myrepo-feat-auth feat/auth

# Create a worktree AND a new branch from the current HEAD
git worktree add -b feat/auth ../myrepo-feat-auth

# Create a worktree AND a new branch from a specific base (typical: from origin/main)
git worktree add -b feat/auth ../myrepo-feat-auth origin/main

# Detached HEAD worktree (read-only inspection of a tag/sha)
git worktree add --detach ../myrepo-inspect v1.4.2

# List worktrees (always do this before remove/move)
git worktree list

# Move a worktree to a new path
git worktree move ../myrepo-feat-auth ../parking/myrepo-feat-auth

# Remove a worktree (refuses if it has uncommitted changes — good)
git worktree remove ../myrepo-feat-auth

# After deleting a worktree directory manually, clean up admin records
git worktree prune
```

### The "I want to work on a new branch" recipe

This is the most common use case. Prescriptive sequence:

```bash
git fetch origin
git worktree add -b feat/<name> ../<repo>-feat-<name> origin/main
cd ../<repo>-feat-<name>
# work, commit, push, open PR
```

When done:

```bash
cd ../<repo>                              # back to main worktree
git worktree remove ../<repo>-feat-<name> # only if the branch is merged or you're done with it
# keep the branch on origin per the user's preference (don't pass --delete-branch)
```

### Pitfalls (these bite people)

- **"Branch already checked out elsewhere"** — you tried to check out a branch that another worktree owns. Fix: `git worktree list` to find which worktree owns it, then either work there or remove that worktree first.
- **Worktree path went stale** — you deleted the directory with `rm -rf` instead of `git worktree remove`. Run `git worktree prune` to clear the admin records.
- **Submodules in worktrees** — submodule paths are *per-worktree* in modern git, but tooling around them is fragile. Re-run `git submodule update --init --recursive` in each new worktree.
- **Hooks** — `core.hooksPath` is shared across worktrees, but local `.git/hooks/` belongs only to the main repo. If hooks live in the main repo's `.git/hooks/`, worktrees won't see them. Use `core.hooksPath = .githooks` (a tracked dir) to make hooks work everywhere.
- **Editor / IDE** — many IDEs misread a worktree's pointer `.git` file. If indexing or git integration breaks, open the worktree path directly as a separate project.
- **`gitignore` / build artifacts** — each worktree has its own `node_modules`, `target/`, `.venv/`, etc. Don't try to share these; bind-mounts and symlinks across worktrees cause more pain than they save.
- **Long-running CI/CD scripts that assume CWD == repo root** — usually fine in a worktree, but scripts that do `git rev-parse --git-dir` and expect `.git/` (not a file pointing elsewhere) may break.
- **`git stash` is per-repo, not per-worktree.** Stashes from worktree A are visible from worktree B. Tag stash messages with the branch name.
- **Locking a worktree you want to keep around** — `git worktree lock <path> --reason "long-running rebuild"` prevents accidental `prune`/`remove`. Unlock with `git worktree unlock <path>`.

### When NOT to use worktrees

- Tiny one-line fix on the current branch — just commit it.
- Switching between branches with no work in flight — `git switch` is simpler.
- If your build system is so slow that double-disk-usage outweighs the parallelism benefit, that's a build problem; worktrees still help, just acknowledge the cost.

---

## Everyday workflows (the prescriptive cheat sheet)

### Starting work

```bash
git fetch origin
git switch -c feat/<name> origin/main   # branch from up-to-date main
```

### Updating a feature branch with latest main

Prefer **rebase** for feature branches that haven't been shared, **merge** for shared branches:

```bash
# Solo branch (preferred — linear history)
git fetch origin
git rebase origin/main

# Shared branch (others may have pulled it)
git fetch origin
git merge origin/main
```

If rebase produces conflicts: resolve, `git add <files>`, `git rebase --continue`. If it's going badly: `git rebase --abort` returns you to where you started — safe.

### Cleaning up a messy branch before PR

Per this repo's `CLAUDE.md`: prefer tangible, meaningful commits. Squash WIP/fixup noise before opening a PR.

```bash
git fetch origin
git rebase -i origin/main
# Mark fixup/WIP commits as `fixup` (or `squash` if you want to combine messages)
```

For the common fixup pattern:

```bash
git commit --fixup=<sha-of-target-commit>
# ...more work...
git rebase -i --autosquash origin/main   # auto-orders and marks fixups
```

### Finding a regression

```bash
git bisect start
git bisect bad                  # current HEAD is broken
git bisect good <known-good-sha>
# git checks out a midpoint; test it
git bisect good   # or: git bisect bad
# repeat until git names the culprit
git bisect reset
```

Automate with a test script:

```bash
git bisect run ./scripts/repro.sh   # exit 0 = good, non-zero = bad, 125 = skip
```

### Bringing one commit over

```bash
git cherry-pick <sha>
# Conflicts? Resolve, then: git cherry-pick --continue
# Want the commit on top with no commit yet: git cherry-pick -n <sha>
```

### Splitting a commit

```bash
git rebase -i <parent-of-commit>
# Mark the commit `edit`
git reset HEAD^               # un-commit, keep changes staged-ish
git add -p                    # interactively stage the first slice
git commit -m "first slice"
git add -p                    # second slice
git commit -m "second slice"
git rebase --continue
```

### Conflict resolution sanity

- `git status` first — it tells you which files are conflicted and what state the operation is in.
- `git diff --name-only --diff-filter=U` lists only the conflicted files.
- After resolving each file: `git add <file>`. **Don't `git add -A`** mid-conflict.
- Use `git mergetool` or your editor's 3-way view if conflicts are non-trivial.
- Abort if it's a mess: `git merge --abort` / `git rebase --abort` / `git cherry-pick --abort`.

### Inspecting history

```bash
git log --oneline --graph --decorate --all       # visual overview
git log -p <file>                                 # changes touching a file
git log -S "<exact string>"                       # commits that added/removed that string
git log -G "<regex>"                              # commits whose diff matches regex
git blame -L 100,150 <file>                       # who touched these lines
git show <sha>                                    # full commit
```

### Working with remotes safely

```bash
git push -u origin <branch>           # first push
git push --force-with-lease           # safer force-push (refuses if remote moved)
# NEVER plain `git push --force` unless the user explicitly asks
```

### Tagging a release

```bash
git tag -a v1.2.3 -m "Release 1.2.3"
git push origin v1.2.3
```

---

## Project conventions that apply here

This repo's `CLAUDE.md` is authoritative — read it. The current rules that affect git workflows:

- **PR merges**: `gh pr merge <n> --merge` (or `--rebase`), **not** squash. Don't pass `--delete-branch`. Only squash or delete-branch when the user explicitly asks for it on a specific PR.
- **Commits**: prefer tangible, self-contained commits with meaningful subjects. Clean up WIP/fixup noise with interactive rebase before opening a PR. Never rewrite already-pushed shared history without authorization.
- **PR descriptions**: do not include a "Test plan" section.

If `AGENTS.md` exists, follow it too — it supplements (and where it conflicts, overrides) generic defaults.

---

## How to respond when invoked

1. **Read the situation first.** `git status`, `git log --oneline -10`, `git worktree list`, and `git stash list` — whichever apply — before recommending a command. Don't recommend operations on imaginary state.
2. **Give the command, then the reason.** Not the other way around. The user wants to act.
3. **Flag the destructive ones.** If your recommendation touches reflog-reachable-only state (reset --hard, branch -D, force-push), state what would be lost and confirm before running it.
4. **One recipe, not five.** If there are multiple ways, pick the one that fits this user/repo and mention the alternative in a single line.
5. **Stop and ask** if the operation is irreversible and the right path isn't obvious from context.
