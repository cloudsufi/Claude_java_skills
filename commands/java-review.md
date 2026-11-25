---
description: Review Java code, a diff or a pull request for correctness, concurrency, errors, performance, tests and data safety
argument-hint: "[file | package | diff | branch | PR number]"
allowed-tools: Read, Grep, Glob, Bash(git diff:*), Bash(git log:*), Bash(git show:*), Bash(git status:*), Bash(git merge-base:*), Bash(git rev-parse:*), Bash(git branch:*), Bash(git fetch:*), Bash(gh pr view:*), Bash(gh pr diff:*), Bash(./gradlew:*), Bash(./mvnw:*)
disable-model-invocation: true
---

# Java code review

**Target:** $ARGUMENTS

Review the target the way a senior Java reviewer would. Every finding cites a real `file:line`, says why it matters, and gives a fix that compiles.

## 1. Resolve the target

| Target | What to review |
|---|---|
| A file, class or package | That code, together with the types it uses and its tests |
| `diff`, or no target at all | The uncommitted changes: `git diff HEAD`, plus new files git doesn't track yet (`git status --porcelain` lists them as `??`). If there are none, the current branch against the default branch (`git diff origin/main...HEAD`, using the repo's real default branch). |
| A branch name | The branch against the default branch, as above |
| A PR number | `gh pr diff` for that number, if the GitHub CLI is installed and signed in. Otherwise ask for the branch or the diff. |

- **Read the reviewed version, not whatever is checked out.** For a branch, read changed files with `git show <branch>:<path>`. For a GitHub PR, fetch it first with `git fetch origin pull/<number>/head`, then read files with `git show FETCH_HEAD:<path>`.
- If the target doesn't exist, the diff is empty, or the folder isn't a git repository, say exactly that and stop. Never review from memory.
- Skip generated sources, vendored code, lockfiles and build output, and say that you skipped them.

## 2. Ground rules

- **Read the repo first:** the build files (`settings.gradle(.kts)`, `build.gradle(.kts)`, `gradle/libs.versions.toml`) and the repo's own rules (`CLAUDE.md`, `README`, `CONTRIBUTING`, the configs under `config/`). Where they differ from this command, the repo wins.
- **Use only what's declared.** Libraries, plugins and Gradle tasks must exist in this build before you cite them. `./gradlew tasks --all` lists the tasks. In a multi-project build, scope tasks to the module, as in `./gradlew :module:check`.
- **Maven repos:** use the Maven equivalents (`./mvnw` or `mvn`) and skip Gradle-only advice, saying so.
- **Label what you couldn't run.** If the build can't run here (no JDK, no network, a blocked repository), say so and give the exact command to run.

## 3. Read the change in context

- Read the whole changed file, the public API it touches, and its tests. Never judge a hunk in isolation.
- Read the repo's written rules: `CLAUDE.md` invariants and editing rules, `CONTRIBUTING`, any "must not" lists, and the PMD, Checkstyle and SpotBugs configs. A change that breaks a written repo rule is a finding, even when no tool catches it.
- If a type or test the change depends on isn't available, mark the findings that depend on it as provisional.

## 4. Checklist

**Correctness and API**
- Does the code do what the change says? Check edge cases: null, empty, boundary and very large inputs.
- `equals`, `hashCode` and `compareTo` are consistent. Values are immutable where possible. Types and visibility are as narrow as possible.
- `Optional` is used only for absent return values, never for fields or parameters.
- An enum is handled by a `switch` expression with no `default`, so a new constant fails to compile. A `switch` statement doesn't get that check, so one over an enum ends with `default -> throw new IllegalStateException(...)` instead of silently skipping new constants.

**Concurrency**
- Shared mutable state is guarded, and state that changes together shares one lock.
- `java.util.concurrent` is used instead of hand-rolled locks; `volatile` and `Atomic*` are used correctly; there are no check-then-act races.
- The interrupt flag is restored after catching `InterruptedException`.

**Resources and errors**
- try-with-resources for every `Closeable` the code owns.
- No swallowed exceptions. A deliberate swallow carries a written reason.
- Exceptions keep their cause and add context. No control flow through exceptions. Entry points fail fast with `Objects.requireNonNull(x, "x")`.

**Performance**
- No N+1 calls, the right collection and algorithm, no needless allocation in hot loops, `StringBuilder` in loops, and expensive resources closed or pooled.

**Readability**
- Names that reveal intent, small methods, no dead code, named constants instead of magic numbers, and comments that explain why. Formatting belongs to the formatter, so leave it out unless no formatter is wired.

**Tests**
- New logic has tests, including negative and edge cases. Tests are deterministic and isolated. A bug fix comes with a regression test. Coverage doesn't drop below the build's gate.

**Security (quick pass)**
- No secrets in code, logs or exceptions. Queries use bound parameters or escaped, allow-listed values. Untrusted XML goes through a hardened parser. TLS verification is never disabled. For a full OWASP pass, suggest the `java-security-review` command to the user.

## 5. Connector data safety (a violation is a Blocker)

These bugs exit 0 and look healthy. Check them whenever the code reads from a source system or stores a position.

- **Ordering:** the checkpoint or position is written only after the host or consumer acknowledged the data. Acknowledge, then checkpoint; never the reverse.
- **Cursors:** the last page doesn't overwrite a good cursor with an empty one. "Nothing new yet" is never returned as "feed finished". An initial-load cursor and a change-feed cursor can't be swapped without an error.
- **Paging:** keyset or token paging, not offset paging over a changing table. End of data comes from the source's reliable end signal (no next token or link, or a short page only where the API guarantees full pages), never from totals or headers the source may misreport.
- **Field names:** if the source silently ignores unknown fields in filters, ordering or projections, every field name sent is checked against the discovered schema.
- **Time:** query literals use the timezone the source interprets them in, values read from rows are normalised to UTC, and window bounds use the source's clock, not the JVM's.
- **Operation type:** insert, update or delete comes from source facts (timestamps, audit records), never from the shape of the payload.
- **Schema:** a schema-change announcement is persisted only after its record is handed over, and survives a restart.
- **Restart:** a restart resumes from the last acknowledged position, and re-reading yields identical records.
- **Configuration:** a key that nothing reads is refused or warned about, never silently ignored.
- **Logging:** no credentials or tokens in logs, exceptions or traces.

## 6. Gates

- Name only the gates the build actually has: formatter, PMD, Checkstyle, SpotBugs, Error Prone, tests, coverage. Suggest any others as "consider adding".
- If the build runs here, run the relevant tasks with `--continue`, so one failing gate doesn't hide the others (for example `./gradlew check --continue`, or `./gradlew :module:check --continue` in a multi-project build), and report failures as findings. If it can't run, list the command instead.

## 7. Output

````markdown
## Review: <target>
**Verdict:** Request changes (2 blockers)

### Blocker
- `src/main/java/.../Foo.java:123`: <what is wrong>. Why: <standard or repo rule>. Fix:
  ```java
  // corrected code
  ```
### Major
### Minor
### Nit
### What's good
````

The verdict is **Approve**, **Approve with nits** or **Request changes**. Put blockers first, keep style nits apart from substance, and say what was done well.

## Avoid these errors

- Cite line numbers from the current file, which is the new side of the diff.
- Suggested fixes must compile. Check the imports and types, and that every API you suggest exists in the library versions the build declares.
- Don't invent commands. Say "run `./gradlew pmdMain`" only if PMD is applied.
- No vague findings such as "could be better". Each one has a location, a reason and a fix.
