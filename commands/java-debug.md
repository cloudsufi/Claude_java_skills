---
description: Debug a Java defect methodically by reproducing it, isolating it, finding the root cause, fixing it and adding a regression test
argument-hint: "[error message | stack trace | failing test | works in X not Y]"
allowed-tools: Read, Grep, Glob, Write, Edit, Bash(./gradlew:*), Bash(./mvnw:*), Bash(git log:*), Bash(git diff:*), Bash(git status:*), Bash(git show:*)
disable-model-invocation: true
---

# Debug a Java defect

**Symptom:** $ARGUMENTS

Work the problem methodically: reproduce, isolate, find the root cause, fix it at the root, and lock it in with a regression test. If no symptom is given, ask for the error message, stack trace or failing test, and stop.

## 1. Preflight

- **Is it real and reproducible?** Confirm the defect from the evidence given. If it can't be reproduced and the report lacks the detail needed (inputs, environment, versions, configuration), say exactly what's missing and stop. Don't "fix" code that isn't proven broken.
- **Whose bug is it?** If the stack trace points into third-party code, say whether the bug is ours (how we call it) or theirs (report it upstream, and work around it) before patching anything.
- **Read the repo's rules** (`CLAUDE.md`, `CONTRIBUTING`): known gotchas are often written down there, and a fix must not break a written invariant.

## 2. Reproduce

1. Capture the exact symptom: the full stack trace (`--stacktrace`), the failing test, or the wrong output.
2. Run the smallest thing that shows it: `./gradlew test --tests 'com.example.FooTest' --info` (scoped to the module in a multi-project build; with Maven, `./mvnw test -Dtest=FooTest`). Test reports are in `build/reports/tests/test/index.html`, and the raw results in `build/test-results/test/`.
3. Write a failing test that reproduces the defect, where possible. That test is the regression test later.

## 3. Isolate

- Narrow to the smallest code path. Read the implicated methods, top of the stack trace first, and the code that produced the bad state, which is often earlier than where it blew up.
- Form one hypothesis at a time, and check it with one probe (an assertion, a log line, a debugger breakpoint, a smaller input). Change one thing at a time, and remove the probes afterwards.
- For a regression ("it used to work"), compare with the last good version: `git log` on the implicated files, and `git diff` between the good and bad commits. A `git bisect` run is an option, with the user's agreement, since it checks out other commits.
- For "works in X, not in Y", list every difference between X and Y: JDK, versions on the classpath, configuration, timezone, locale, data, file paths, permissions.

## 4. Common causes, and where to look

| Symptom | Usual cause | Check |
|---|---|---|
| `NoSuchMethodError`, `NoClassDefFoundError`, `ClassNotFoundException` at runtime | Two versions of a library on the classpath, or a dependency missing at runtime | `./gradlew dependencyInsight --dependency <name> --configuration runtimeClasspath` |
| `UnsupportedClassVersionError` | Compiled for a newer Java than the one running it | The toolchain, `JAVA_HOME`, and the runtime's `java -version` |
| "Could not find group:artifact:version" | An upstream module wasn't published, or was published before its last change (`mavenLocal()` chains) | Republish upstream, then rebuild |
| Tests pass but none ran (Gradle 8, with a "No test executed" deprecation warning), or the build fails because no test ran (Gradle 9) | `useJUnitPlatform()` missing from the test task | The test count in the report |
| `UnnecessaryStubbingException` | A Mockito stub the test never uses (strict stubs) | Remove it, or mark it `lenient()` if genuinely conditional |
| "Java NN is not supported" from Byte Buddy | Mockito too old for the JDK | Upgrade Mockito |
| "Task with name 'x' not found" | The build wires a task whose plugin isn't applied | Use `tasks.matching { it.name == 'x' }.configureEach { ... }` |
| PMD "Unable to find referenced rule" | A PMD 6 ruleset on PMD 7 | Update the ruleset to PMD 7 names |
| `javadoc` fails on `<p/>`, a raw `<`, a heading or a `@param` | Doclint errors | Fix the comment; don't turn doclint off |
| `IllegalAccessError` from google-java-format | Spotless too old for the JDK | Upgrade the Spotless plugin |
| A flaky test | Shared state, test order, real time, real network | Run the test alone and repeatedly; inject the clock |
| Wrong times or off-by-hours data | Timezone mismatch between the source, the JVM and the stored values | Where literals are rendered, and where values are normalised to UTC |
| Missing or duplicated records in a connector | Offset paging over changing data, a cursor lost on the last page, a checkpoint written before acknowledgement | The paging, cursor and checkpoint code paths |
| A filter or ordering that seems to be ignored | The source silently drops unknown field names | Check every field name against the discovered schema |

## 5. Root cause, fix and lock-in

1. **State the root cause, not the symptom:** why it happens (a null contract, a race, an off-by-one, a wrong config, a resource leak, an encoding or timezone issue), with the evidence.
2. **Fix at the root, minimally.** No unrelated clean-ups in the same change.
3. **Run the failing test** (now green), then the module's tests and the gates the build has (`./gradlew check`).
4. **Keep the regression test,** named for the bug and the date.
5. **Look for siblings:** search for the same flaw elsewhere (`Grep` for the pattern) and list the spots, fixing them only if the user agrees.

## Output

1. The symptom, and how it was reproduced.
2. The root cause, with evidence (`file:line`).
3. The fix, as a diff.
4. The regression test, and the test results before and after.
5. Related spots with the same flaw.

If the build can't run here, say that the fix is unverified, and give the exact commands to reproduce and to verify it.
