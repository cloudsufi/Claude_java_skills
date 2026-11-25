---
description: Write a commit message for the staged changes, following the repo's commit style, and commit only if asked
argument-hint: "[extra context, e.g. a ticket id] [commit]"
allowed-tools: Read, Grep, Glob, Bash(git status:*), Bash(git diff:*), Bash(git log:*), Bash(git branch:*), Bash(git rev-parse:*), Bash(git show:*)
disable-model-invocation: true
---

# Generate a commit message

**Input:** $ARGUMENTS

Write one commit message that says what changed and why, in the style this repository already uses. Anything in the input is extra context (a ticket id, the reason for the change). If the input contains the word `commit`, make the commit after showing the message; otherwise only show the message.

## 1. Find what will be committed

1. Check this is a git repository (`git rev-parse --is-inside-work-tree`). If it isn't, say so and stop.
2. Run `git status --porcelain` and `git diff --cached --stat`.
   - **Something is staged:** the message describes the staged changes only (`git diff --cached`). Mention, in one line, any unstaged or untracked files that won't be included.
   - **Nothing staged, but there are changes:** say that nothing is staged, list the changed and untracked files, and write the message for all of them as a draft. Never stage files yourself unless the user asks; suggest `git add <files>` instead.
   - **No changes at all:** say there's nothing to commit, and stop.
3. Read the full diff, not just the file list. For a large diff, read the hunks of every file, starting with the source code, then tests, build files and docs. Skip lockfiles and generated files, but count them.

## 2. Learn the repo's commit style

- Read the last 20 subjects: `git log -20 --format=%s`.
- **Conventional Commits** (`feat:`, `fix(core):`, `chore:`...) in most of them: use the same format, picking the type from the change (`feat`, `fix`, `refactor`, `test`, `docs`, `build`, `ci`, `perf`, `chore`) and a scope only if the repo uses scopes.
- **Ticket ids** (such as `ABC-123`) in the subjects: include the ticket the same way. Take it from the input or the branch name (`git branch --show-current`, e.g. `feature/ABC-123-paging`). If there is none, leave it out; never invent one.
- **Otherwise:** a plain imperative subject, as in "Add keyset paging to the record reader".
- Follow any commit rules written in `CONTRIBUTING`, `CLAUDE.md` or a commit template (`git config commit.template`).

## 3. Write the message

- **Subject:** imperative mood ("Add", "Fix", not "Added"), about 50 characters and never over 72, no trailing period. It names the most important change.
- **Blank line,** then a **body** wrapped at 72 characters that explains why the change was made and what it affects. Use short bullets for several related changes. Mention behaviour changes, new configuration keys, migrations and anything reviewers or operators must know.
- **Footer**, only when it applies: `BREAKING CHANGE: ...` for incompatible changes (Conventional Commits), and the repo's ticket line (`Refs: ABC-123`) if it uses one.
- A tiny change (a typo, a version bump) gets a subject only.
- If the staged changes mix unrelated work (a feature plus an unrelated refactor), say so and suggest splitting them into separate commits, with a message for each.
- Describe only what the diff shows. Don't claim tests pass unless you ran them.

## 4. Output

Show the message in a code block, ready to copy. Then show the command to use it:

```bash
git commit -F- <<'MSG'
<the message>
MSG
```

If the input asked you to `commit`, run that command after showing the message (Claude Code asks for permission first). Never use `--no-verify`, never amend an existing commit, and never push, unless the user asks for it.

## Avoid these errors

- If a pre-commit hook fails, report its output; fix the cause only if the user asks, and never bypass the hook.
- Don't mention files that aren't in the commit, and don't paste secrets from the diff into the message.
