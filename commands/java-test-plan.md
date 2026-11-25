---
description: Plan how to test a Java feature or module, with layers, risk order, fakes, coverage and mutation gates, and CI
argument-hint: "[feature | module | path]"
allowed-tools: Read, Grep, Glob, Bash(./gradlew tasks:*), Bash(./gradlew help:*), Bash(git log:*), Bash(git status:*)
disable-model-invocation: true
---

# Java test plan

**Target:** $ARGUMENTS

Produce a test plan that fits the tools this repo already uses. If no target is given, plan for the module in the current directory. To write the tests themselves afterwards, suggest the `java-test` command to the user.

## 1. Take stock first (read the repo; don't guess)

- **Test frameworks** on the test classpath: JUnit 4 or 5, AssertJ, Mockito, Testcontainers.
- **Fakes and fixtures:** `src/testFixtures`, `testkit` packages, `Fake*` and `InMemory*` classes.
- **Today's split** between unit and integration tests: `*IT` classes, JUnit tags, separate source sets or tasks.
- **Coverage:** the JaCoCo gate and its thresholds, and today's numbers if a report exists under `build/reports/jacoco/`.
- **Mutation testing:** whether PIT is wired.
- **CI:** `.github/workflows/`, `Jenkinsfile`, `.gitlab-ci.yml` or `azure-pipelines.yml`.
- **Existing strategy:** a test section in a design doc, or test rules in `CLAUDE.md` or `CONTRIBUTING`. If one exists, propose changes to it rather than a competing plan.

Keep it proportionate. If the target is pure data holders or trivial delegation, say where the real risk lives instead. For a small change, a three-line plan is fine.

## 2. Build the plan

1. **Map the surface:** public behaviours, branches, edge cases, error paths, concurrency, and integration points (database, HTTP, queues, files). Rank them by risk, starting with failures that are both silent and costly.
2. **Place each item at the right layer,** aiming for roughly 80% unit, 20% integration and a few end-to-end tests.
   - **Unit:** pure logic in isolation. Replace collaborators with fakes (preferred where the repo owns the interface) or mocks. Fast, numerous, deterministic.
   - **Integration:** real infrastructure, such as a database, a broker, the filesystem or the host's own driver, with no mock of the thing under integration. One shared instance per test class.
   - **End-to-end or contract:** a thin layer for the happy path and the key contracts.
3. **Decide what to test, and what not to.**
   - *Do:* business rules, every guard, boundaries, null and empty input, error paths, serialization round trips, concurrency invariants, restart and resume behaviour.
   - *Don't:* trivial getters, the framework itself, or 100% coverage for its own sake.
4. **Pick assertions and data:** the repo's assertion library, parameterized tests for tables, and builders or fixtures instead of copy-paste.
5. **Set the gates from today's numbers,** then raise them deliberately. Use JaCoCo `jacocoTestCoverageVerification` with instruction and branch minimums wired into `check`, and mutation testing (PIT) on critical packages. Coverage proves code ran; mutation proves the assertions would catch a regression.
6. **Wire CI** in the system the repo already uses: unit tests on every push, integration tests on pull requests, a failed build below the thresholds, and published reports.

## 3. Gate snippets (Groovy DSL; translate if the repo uses Kotlin DSL)

JaCoCo is a core plugin. Checked on Gradle 8.14:

```groovy
jacoco { toolVersion = '0.8.15' }      // Java 21 needs 0.8.11 or later
tasks.named('jacocoTestCoverageVerification') {
  dependsOn tasks.named('test')
  violationRules {
    rule {
      limit { counter = 'INSTRUCTION'; minimum = 0.75 }
      limit { counter = 'BRANCH'; minimum = 0.60 }
    }
  }
}
tasks.named('check') { dependsOn tasks.named('jacocoTestCoverageVerification') }
```

Splitting integration tests out by JUnit tag. Checked on Gradle 8.14:

```groovy
tasks.named('test') { useJUnitPlatform { excludeTags 'integration' } }
tasks.register('integrationTest', Test) {
  description = 'Runs tests tagged "integration".'
  group = 'verification'
  testClassesDirs = sourceSets.test.output.classesDirs
  classpath = sourceSets.test.runtimeClasspath
  useJUnitPlatform { includeTags 'integration' }
  shouldRunAfter tasks.named('test')
}
tasks.named('check') { dependsOn tasks.named('integrationTest') }
```

Or by class name, for repos that name integration tests `*IT`. Checked on Gradle 8.14: each class runs in exactly one task, and the build still passes before the first `*IT` class exists.

```groovy
tasks.named('test') {
  useJUnitPlatform()
  filter { excludeTestsMatching '*IT' }
}
tasks.register('integrationTest', Test) {
  description = 'Runs the *IT test classes.'
  group = 'verification'
  testClassesDirs = sourceSets.test.output.classesDirs
  classpath = sourceSets.test.runtimeClasspath
  useJUnitPlatform()
  filter { includeTestsMatching '*IT'; failOnNoMatchingTests = false }
  shouldRunAfter tasks.named('test')
}
tasks.named('check') { dependsOn tasks.named('integrationTest') }
```

Many connector repos keep `*IT` classes in the same source set and the same `test` task instead. That's fine; follow what the repo does.

PIT needs the community plugin `info.solidsoft.pitest`. Take its current version and the JUnit 5 plugin version from its docs, and add it only when asked. Without `junit5PluginVersion`, PIT finds no JUnit 5 tests. Scope `targetClasses` to the critical packages, set `mutationThreshold`, and set `timestampedReports = false` so the report path is stable.

## In connector repos

- **Rank silent data-loss risks first.** These failures exit 0 and look healthy, and each deserves a named test:
  - a checkpoint written before the host acknowledged;
  - a cursor lost on the last page;
  - offset paging that skips rows;
  - a timezone shift in a watermark;
  - deletes that are never observed;
  - a schema change mid-run;
  - an idle feed mistaken for the end;
  - duplicates after a restart.
- **Prefer a fake source to mocking the HTTP client.** The fake should reproduce the source's real quirks. Coverage earned by mocking the transport measures the mock.
- **Use three layers:** unit tests for mapping and decision logic; integration tests through the host's real driver, where the host provides one, for ordering, restart and crash-resume; and a live verification run against a real instance for facts only the instance has (timezone, audit lag, permissions, OAuth). The live run needs credentials and writes data, so it runs only on request, and cleans up afterwards.
- **Set coverage floors by a module's role.** Transport plumbing that is verified live can carry a lower unit-coverage floor than the core contract. Write the reason next to the number.
- **Mutation-check by hand where PIT isn't wired:** break the guard, watch the test fail, then restore it.

## Output

A markdown plan with these sections:
1. Scope and current state (frameworks, fixtures, coverage today, CI).
2. Risks, ranked.
3. A table: item, layer, how it's tested, fixture or fake used.
4. The first five tests to write.
5. Gates: current value, target value, and the reason for each.
6. CI wiring.
7. What won't be tested, and why.

## Avoid these errors

- A coverage floor above today's coverage fails `check` on the next build. Measure first.
- A coverage exclusion has to be applied to both the report and the verification task, or the two numbers disagree.
- Testcontainers needs Docker on every machine and CI agent that runs those tests. Guard the tests with `disabledWithoutDocker`, or keep them out of the default `test` task.
- A custom `Test` task needs `testClassesDirs` and `classpath`; Gradle 9 rejects one without them.
- PIT is slow. Run it on demand or nightly, not on every build.
- Never put a placeholder version in a build file. If you can't look a version up, leave the gate out and say what to add.
