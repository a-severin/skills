---
name: clean-local-branches
description: >
  Delete local git branches that have no counterpart on any remote. Prunes
  stale remote-tracking refs first, never touches the current branch, the
  default branch, or branches checked out in other worktrees, records every
  deleted tip SHA for recovery, and reports what it kept and why. Use when the
  user says "clean up local branches", "delete branches without a remote",
  "prune local branches", "почисти локальные ветки", "удали ветки без remote",
  or asks to tidy the branch list after merged or abandoned work. Do NOT use
  to delete one specific branch the user names, and do NOT delete remote
  branches — this skill only removes local refs.
modeSlugs:
  - orchestrator
  - code
---

# clean-local-branches

Remove **orphaned** local branches: branches whose counterpart no longer exists
on any remote. The criterion is remote existence, not upstream-tracking
configuration — `git branch -vv` showing `[gone]` is a symptom, not the test,
because a branch can have no upstream configured and still exist on the remote.

## When to use

- The branch list has accumulated work that was merged or abandoned upstream.
- The user wants a clean `git branch` output.
- Remote branches were deleted after merge and the local copies linger.

Do not use this to delete a branch the user explicitly names (just delete that
one), and do not use it to delete anything on the remote.

## Arguments

`$ARGUMENTS` is optional. Recognise:

- `dry-run`, `preview`, `show`, `покажи`, `--dry-run` — print the plan and stop;
  delete nothing.
- a remote name (`origin`, `upstream`, …) — consider only that remote when
  deciding whether a branch is orphaned. Default: all remotes.

Any other text: treat as context and fall back to the default (all remotes,
delete).

## Procedure

### 1. Establish context

```bash
git rev-parse --is-inside-work-tree
git symbolic-ref --short HEAD                       # current branch — never delete
git worktree list --porcelain | awk '/^branch /{sub("refs/heads/","",$2); print $2}'  # never delete
git for-each-ref --format='%(refname:short)' refs/heads
```

Refuse to continue outside a work tree. Build three protection sets:
`current`, `worktrees`, and `default` (see step 3).

### 2. Refresh the remote-tracking refs

```bash
git fetch --prune --all
```

`--prune` is what makes a deleted remote branch disappear locally, so the
existence test below is accurate. Without it you would delete nothing or,
worse, judge from stale refs.

If the network is unavailable, say so and stop — do not proceed on stale data.

### 3. Decide the default branch

```bash
git symbolic-ref -q refs/remotes/origin/HEAD | sed 's#^refs/remotes/origin/##'
```

If that prints nothing, fall back to `main` or `master`, whichever exists as a
remote-tracking ref. The default branch is always protected.

### 4. Classify every local branch

A local branch is **kept** if any of these hold:

- it is the current branch (`git symbolic-ref --short HEAD`);
- it appears in `git worktree list --porcelain`;
- it is the default branch;
- a remote-tracking ref `<remote>/<branch>` exists for any considered remote.

Otherwise it is **orphaned** and eligible for deletion.

Get the set of names that exist on the considered remotes with:

```bash
git for-each-ref --format='%(refname:lstrip=2)' refs/remotes \
  | grep -v '/HEAD$' | cut -d/ -f2- | sort -u
```

(`lstrip=2` turns `refs/remotes/origin/foo` into `origin/foo`; stripping the
first path component leaves the branch name. Skip `<remote>/HEAD`.)

Restrict to one remote when the argument names one.

### 5. Preview — always show this before deleting

For every orphaned branch, print its tip and how much unique work it holds:

```bash
git log -1 --format='%h %s' "<branch>"
git rev-list --count "<default-remote>..<branch>"   # commits not in the default branch
```

If the count is zero the work is already in the default branch and deletion is
lossless. If it is non-zero, say so explicitly — that commit is about to become
unreachable except through the reflog.

If `$ARGUMENTS` asked for a dry run, stop here and report the table.

### 6. Delete

```bash
git branch -D "<branch>"
```

`-D` (force) is correct here: `-d` refuses branches whose work is not merged and
would abort the batch. The orphan criterion in step 4 is what makes the force
safe — re-verify it per branch rather than assuming.

Skip any branch that moved on to have a remote since step 2 (a concurrent
`git fetch`) by re-testing existence immediately before deleting.

### 7. Report

Produce three sections:

1. **Deleted** — branch, tip SHA, commits not in the default branch. Keep the
   SHA: it is the recovery key.
2. **Kept** — branch and the reason (`has origin/…`, `current`, `worktree`,
   `default`).
3. **Recovery** — the exact command, with a real SHA:

   ```bash
   git branch <name> <sha>
   ```

   Note that reflog also retains the tips until it is pruned, so a recoverable
   copy exists even after `git branch -D`.

## Pitfalls

- **Do not decide from `[gone]` alone.** It only means a configured upstream
  was deleted; branches with no upstream never show it. Compare against
  `refs/remotes/*` (step 4).
- **Do not use `git remote prune` as the cleanup.** It removes remote-tracking
  refs, never local branches.
- **Do not run `git branch -d` on a batch and stop on the first refusal.**
  Decide per branch; unmerged orphans need `-D`.
- **Never delete the default branch, the current branch, or a branch checked
  out in another worktree.** `git branch -D` refuses the checked-out ones
  anyway, but skipping them deliberately keeps the report honest.
- **`git branch` names are unique per local repo, not per remote.** With
  several remotes, keep the branch if *any* considered remote has it.
- **Do not delete remote branches.** If the user asks, that is
  `git push <remote> --delete <branch>` — a separate, confirmable decision.
