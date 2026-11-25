---
description: Find untested risky Java code with JaCoCo, list the missing test cases, and set coverage and mutation gates
argument-hint: "[module | package]"
allowed-tools: Read, Grep, Glob, Write, Edit, Bash(./gradlew:*), Bash(./mvnw:*)
disable-model-invocation: true
---

# Java coverage gaps

**Target:** $ARGUMENTS

Find the code that runs untested where it matters, list the missing cases, and make sure the build enforces a sensible floor. If no target is given, use the module in the current directory.

## 1. Preflight

- **Read the build first.** Is the `jacoco` plugin applied? Are the XML and HTML reports enabled? Is `jacocoTestCoverageVerification` wired into `check`, and with what minimums? Is PIT (`info.solidsoft.pitest`) applied? In a Maven repo, look for `jacoco-maven-plugin` and `pitest-maven` in `pom.xml` instead.
- **No JaCoCo:** say so. Show the snippet in step 4, but add it to the build only if the user agrees. Meanwhile, find gaps by reading the code against its tests.
- **No tests at all:** say the first step is writing tests (the `java-test` command), not gap analysis.
- **Already at the agreed floor:** report the numbers as evidence and stop. Suggest mutation testing or specific missing error paths only where they're real gaps.
- **Can't run the build here:** say so, and give the commands below for the user to run.

## 2. Measure

Run the tests and the report, scoped to the module in a multi-project build:

```bash
./gradlew test jacocoTestReport
```

The reports land in `build/reports/jacoco/test/`: `jacocoTestReport.xml`, and `html/index.html`. If `python3` is available, this lists the 20 classes with the most missed branches from the XML report (pass the report's real path):

```bash
python3 - build/reports/jacoco/test/jacocoTestReport.xml <<'EOF'
import sys
import xml.etree.ElementTree as ET

root = ET.parse(sys.argv[1]).getroot()
rows = []
for pkg in root.iter("package"):
    for cls in pkg.findall("class"):
        counters = {c.get("type"): (int(c.get("missed")), int(c.get("covered"))) for c in cls.findall("counter")}
        b_missed, b_covered = counters.get("BRANCH", (0, 0))
        l_missed, l_covered = counters.get("LINE", (0, 0))
        rows.append((b_missed, l_missed, cls.get("name"), b_missed + b_covered, l_missed + l_covered))
rows.sort(reverse=True)
print("missed branches | missed lines | class")
for b_missed, l_missed, name, b_total, l_total in rows[:20]:
    print(f"{b_missed:>5} of {b_total:<5} | {l_missed:>5} of {l_total:<5} | {name}")
EOF
```

If the XML report isn't enabled, read the HTML report instead, or enable `xml.required = true` with the user's agreement.

## 3. Turn gaps into missing cases

- **Rank by risk, not by percentage.** Business rules, guards, error handling, parsing, cursor and checkpoint logic come first. Ignore trivial accessors, generated code and `toString`.
- For each risky class, open its source next to the report's line view and list the missing cases: the negative path, the boundary, the error branch, the null or empty input, the restart path.
- Write the tests only if the user asks, in the repo's existing test style. Otherwise suggest the `java-test` command for them.

## 4. Gates

Set floors from the measured numbers, never above them, or `check` fails on the next build. JaCoCo is a core plugin; this was checked on Gradle 8.14:

```groovy
jacoco { toolVersion = '0.8.15' }      // Java 21 needs 0.8.11 or later
tasks.named('jacocoTestReport') {
  dependsOn tasks.named('test')
  reports { xml.required = true; html.required = true }
}
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

- Replace `0.75` and `0.60` with today's measured values, rounded down, and write the reason next to each number.
- A module whose plumbing is verified live can carry a lower floor than the core logic. Say so next to the number.
- If you exclude classes from coverage, apply the same exclusion to both the report and the verification task, or the two numbers disagree.

## 5. Mutation testing

Coverage proves code ran; mutation testing proves the assertions would catch a regression.

- **PIT already applied:** run `./gradlew pitest`, and report the mutation score and the surviving mutants in the critical classes.
- **PIT not applied:** recommend it for the critical packages only, and add it only when asked. It needs the plugin `info.solidsoft.pitest` and its `junit5PluginVersion` setting (without it, PIT finds no JUnit 5 tests). Take both versions from the plugin's docs. PIT is slow, so run it on demand or nightly.
- **Meanwhile, mutate by hand:** break one critical guard, confirm a test fails, then restore the code exactly.

## Output

1. Current numbers: instruction, branch and line coverage for the target.
2. The top risky gaps, each with `file:line` and the missing case.
3. The gate change, if any, as a diff with the reason for each threshold.
4. The mutation result, or the recommendation.

## Avoid these errors

- Run only tasks that exist: `jacocoTestReport` needs the `jacoco` plugin, and `pitest` needs the PIT plugin. Check with `./gradlew tasks --all`.
- Don't chase 100%. Target meaningful coverage of the logic that can fail silently.
- Never put a placeholder version in a build file. If you can't look one up, say what to add instead.
