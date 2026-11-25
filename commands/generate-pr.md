---
description: Write a pull request title and description for the current branch against a target branch (main by default)
argument-hint: "[target branch, default main] [create]"
allowed-tools: Read, Grep, Glob, Bash(git status:*), Bash(git diff:*), Bash(git log:*), Bash(git branch:*), Bash(git rev-parse:*), Bash(git merge-base:*), Bash(git fetch:*), Bash(git show:*), Bash(git symbolic-ref:*), Bash(git ls-remote:*), Bash(gh pr view:*), Bash(gh pr list:*), Bash(gh auth status:*)
disable-model-invocation: true
---

# Generate a pull request

**Input:** $ARGUMENTS

Write a PR title and description for everything on the current branch that isn't on the target branch yet. The first word of the input that isn't `create` is the target branch; with no target, use `main`. If the input contains `create`, also open the PR on GitHub after showing it (section 5); otherwise only show it.

## 1. Resolve the branches

1. Check this is a git repository (`git rev-parse --is-inside-work-tree`). If not, say so and stop.
2. **Current branch:** `git branch --show-current`. If it's empty (detached HEAD) or it *is* the target branch, say so and stop: a PR needs a feature branch.
3. **Target branch:**
   - Refresh it if there's a remote: `git fetch origin <target>`. If the fetch fails (offline, no access), continue with what's local and say the comparison may be out of date.
   - Compare against `origin/<target>` if it exists (`git rev-parse --verify --quiet origin/<target>`), otherwise the local `<target>`.
   - If no target was given and `main` doesn't exist, use the remote's default branch (`git symbolic-ref --short refs/remotes/origin/HEAD` prints it as `origin/<name>`), or `master` if that exists, and say which one you used.
   - If the named target doesn't exist at all, say so, list the branches that do (`git branch -a`), and stop.
4. **Uncommitted work:** if `git status --porcelain` shows changes, say they won't be in the PR, and suggest the `generate-commit` command first.

## 2. Gather the change

- Commits: `git log --format='%h %s' <base>..HEAD` (where `<base>` is the target from step 1). If there are none, say the branch has nothing new compared with the target, and stop.
- Size: `git diff --stat <base>...HEAD`. The three-dot form compares with the merge base, so changes made on the target since the branch started don't show up as yours.
- Content: read `git diff <base>...HEAD` file by file, starting with the source code, then tests, build files, config and docs. Skip lockfiles and generated files, but count them.
- Ticket ids: from the branch name (for example `feature/ABC-123-paging`), the commit messages, or the input. Never invent one.
- Merge conflicts: if `git merge-base` shows the branch is far behind the target, mention that a rebase or merge may be needed before review.

## 3. Use the repo's PR template

Look for a template: `.github/pull_request_template.md`, `.github/PULL_REQUEST_TEMPLATE.md`, `.github/PULL_REQUEST_TEMPLATE/`, `docs/pull_request_template.md`, or `PULL_REQUEST_TEMPLATE.md` at the root. If there is one, fill in its sections exactly and keep its checkboxes, ticking only what you verified. If there are several, ask which one fits. If there's none, use the default below. Also follow any PR rules in `CONTRIBUTING` or `CLAUDE.md`.

## 4. Write it

**Title:** under about 70 characters, imperative, saying what the PR does. Match the repo's style: Conventional Commits prefixes or ticket ids if recent PRs or commits use them.

**Default description:**

```markdown
## Summary
What this PR does and why, in two or three sentences. Link the ticket if there is one.

## Changes
- The main changes, grouped by module or concern, in terms a reviewer can check.

## How it was tested
- The tests added or changed, and what they cover.
- Commands run and their results (only ones actually run; otherwise say "not run").

## Risks and rollout
- Behaviour changes, new or changed configuration keys, migrations, compatibility.
- For connector code: effects on checkpoints, cursors, paging, restart behaviour or delivery guarantees, or "none".

## Notes for reviewers
- Where to start reading, and anything deliberately left out of scope.
```

- Leave out sections that would only say "n/a", except "How it was tested".
- Describe only what the diff and commits show. Don't claim tests or builds passed unless you ran them, or the commits clearly record it.
- For a large PR (roughly 30 files or 1,000 changed lines and up), say so and suggest how it could be split.
- Never paste secrets or credentials from the diff into the description.

## 5. Open it (only if the input contains `create`)

1. Check the GitHub CLI is installed and signed in (`gh auth status`). If not, say so and give the title and body for the user to paste into the web page instead.
2. Check no PR is already open for this branch (`gh pr view`). If one is, show its link and offer the new description as an edit instead.
3. The branch must be on the remote. If it isn't pushed, or it's ahead of its upstream, ask the user before pushing (`git push -u origin <branch>`). Never force-push.
4. Create the PR as a draft unless the user asked otherwise:
   ```bash
   gh pr create --draft --base <target> --head <branch> --title "<title>" --body-file - <<'BODY'
   <the description>
   BODY
   ```
5. Show the PR link.

## Output

The title, then the description in a markdown code block so it can be copied as-is, then a one-line note of anything the user should check (uncommitted work left out, an out-of-date target, a branch behind the target, a PR too large).

## Avoid these errors

- Compare with three dots (`<base>...HEAD`) for the diff, and two dots (`<base>..HEAD`) for the commit list.
- Never push, force-push, rebase or change branches unless the user asks.
- If the repository isn't on GitHub, or `gh` isn't available, only produce the text; don't try other tools to open the PR.
