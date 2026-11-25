---
description: Audit, fix or set up Java coding standards with Spotless, PMD, Checkstyle, SpotBugs or Error Prone as build gates
argument-hint: "[audit | fix | setup] [repo | module]"
allowed-tools: Read, Grep, Glob, Write, Edit, Bash(./gradlew:*), Bash(./mvnw:*)
disable-model-invocation: true
---

# Java coding standards

**Target:** $ARGUMENTS

Pick the mode from the target. With no mode, **audit** the module in the current directory.

| Mode | What to do |
|---|---|
| `audit` | Run the gates the build already has, group the findings by rule, and report fixes. Change nothing. |
| `fix` | The build fails on a formatter or lint rule: fix the code until the gates pass. |
| `setup` | Add missing gates, only the ones the user asked for, wired so `./gradlew check` fails on any violation. |

## 1. Preflight

- **What the repo already enforces is the standard.** Find:
  - the formatter (a `spotless {}` block, `.editorconfig`);
  - rulesets under `config/` (`config/pmd/`, `config/checkstyle/`, `config/spotbugs/`);
  - gate plugins in the build file, and their versions;
  - written rules in `CLAUDE.md`, `CONTRIBUTING` or `README`.
- **Run only tasks that exist.** `spotlessCheck` exists only with Spotless, `checkstyleMain` only with Checkstyle, `pmdMain` only with PMD. Check with `./gradlew tasks --all`. In a multi-project build, scope them, as in `./gradlew :module:pmdMain`.
- **Measure before enforcing.** Before turning a new gate on for existing code, run it once in report mode (`ignoreFailures = true`) to see the backlog. A gate with hundreds of findings breaks every build until they're fixed or baselined.
- **Don't add tools nobody asked for.** In `audit` and `fix` modes, suggest missing gates as "consider adding".
- **Label what you couldn't run.** If the wrapper, a JDK or the plugin downloads aren't available here, deliver the config and fixes as unverified, with the command that checks them.

## 2. The standard (what good looks like)

- **Format.** A tool owns layout, so nobody argues about formatting. Usually that's google-java-format through Spotless, plus `removeUnusedImports`, `trimTrailingWhitespace` and `endWithNewline`, and a license header when the repo has a header file.
- **Naming and structure.** Checkstyle with Google's ruleset, where the repo uses Checkstyle.
- **Smells and bugs.** PMD categories (best practices, error-prone, design, performance, multithreading, security, code style), with noisy rules excluded and every exclusion commented. SpotBugs for bug patterns. Error Prone as a compiler plugin.
- **Idioms (Effective Java):**
  - classes `final` unless built for extension, and visibility as narrow as possible;
  - records for values, with defensive copies (`List.copyOf`, `Map.copyOf`) on the way in and out;
  - builders for types with many fields, and static factories (`of`, `from`, `parse`);
  - `Optional` only for absent return values;
  - `equals` and `hashCode` overridden together;
  - fail fast with `Objects.requireNonNull(x, "x")`, and refuse ambiguous input loudly, naming the key and the value;
  - enums handled by `switch` expressions with no `default`, so a new constant fails to compile (a `switch` statement doesn't get that check, so one over an enum ends with `default -> throw new IllegalStateException(...)`);
  - `java.util.concurrent` instead of hand-rolled locks, and the interrupt flag restored after catching `InterruptedException`;
  - try-with-resources for every owned `Closeable`, and no empty `catch` without a written reason;
  - logging through the project's facade (usually SLF4J), never `System.out` outside command-line launchers;
  - config keys and metadata names declared once, as constants;
  - comments that explain why, not what.
- **Project policy rules.** Some repos enforce these; follow them wherever the ruleset does: final method parameters (PMD `MethodArgumentCouldBeFinal`), sometimes on test sources too; no reassigned parameters (`AvoidReassigningParameters`); and suppressions only as `@SuppressWarnings("PMD.RuleName") // reason`, at the narrowest scope.

## 3. Setup snippets (Groovy DSL; translate if the repo uses Kotlin DSL)

```groovy
plugins {
  id 'pmd'
  id 'checkstyle'
  id 'com.diffplug.spotless' version '8.9.0'    // known-good with Gradle 8.10 and Java 21
}
pmd {
  toolVersion = '7.26.0'
  ignoreFailures = false
  ruleSetFiles = files('config/pmd/pmd-ruleset.xml')
  ruleSets = []                                  // otherwise Gradle's default ruleset runs too
}
checkstyle {
  toolVersion = '…'        // fill in a current release; take google_checks.xml from the same release
  configFile = file('config/checkstyle/checkstyle.xml')
  maxWarnings = 0
}
spotless {
  java { googleJavaFormat(); removeUnusedImports(); trimTrailingWhitespace(); endWithNewline() }
}
```

Replace the `…` with a real Checkstyle release before saving the file. If you can't look one up, leave Checkstyle out and say what to add.

Other gates, only when asked:
- **SpotBugs:** plugin `id 'com.github.spotbugs'` (there is no plugin called `spotbugs`), with `spotbugs { ignoreFailures = false }`. Keep its config minimal, because the types of `effort` and `reportLevel` changed between plugin 5.x and 6.x.
- **Error Prone:** plugin `id 'net.ltgt.errorprone'`, plus `dependencies { errorprone 'com.google.errorprone:error_prone_core:<current version>' }`. Without that dependency, compilation fails. Turn it on after a clean baseline.
- **Format, then lint, on every compile** (only if the team wants it, because it rewrites sources during the build, and every task that compiles also runs PMD):
  ```groovy
  tasks.named('compileJava') { dependsOn 'spotlessApply'; finalizedBy 'pmdMain' }
  tasks.named('pmdMain') { mustRunAfter 'spotlessApply', 'compileJava' }
  ```

### Starter PMD 7 ruleset (`config/pmd/pmd-ruleset.xml`)

It mirrors a ruleset that connector modules run on PMD 7.26, so every reference and exclusion below exists in PMD 7. It keeps `MethodArgumentCouldBeFinal` on as policy; on existing code that rule produces most of the first findings, so baseline them before enforcing.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ruleset name="Project rules"
         xmlns="http://pmd.sourceforge.net/ruleset/2.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://pmd.sourceforge.net/ruleset/2.0.0 https://pmd.sourceforge.io/ruleset_2_0_0.xsd">
  <description>Project rules. Each block says why its exclusions exist.</description>

  <!-- Error-prone: keep almost all of it. -->
  <rule ref="category/java/errorprone.xml">
    <exclude name="AvoidLiteralsInIfCondition"/>
    <exclude name="AvoidFieldNameMatchingMethodName"/>
    <exclude name="AvoidDuplicateLiterals"/>
    <exclude name="UseProperClassLoader"/>
  </rule>

  <!-- Best practices: drop the logging and comparison-order opinions. -->
  <rule ref="category/java/bestpractices.xml">
    <exclude name="GuardLogStatement"/>
    <exclude name="LiteralsFirstInComparisons"/>
  </rule>

  <rule ref="category/java/performance.xml">
    <exclude name="AvoidInstantiatingObjectsInLoops"/>
  </rule>

  <!-- Multithreading: keep the checks, drop the "never use threads/volatile" bans. -->
  <rule ref="category/java/multithreading.xml">
    <exclude name="UseConcurrentHashMap"/>
    <exclude name="DoNotUseThreads"/>
    <exclude name="AvoidUsingVolatile"/>
    <exclude name="AvoidSynchronizedStatement"/>
    <exclude name="AvoidSynchronizedAtMethodLevel"/>
  </rule>

  <rule ref="category/java/security.xml"/>

  <!-- Design: keep it, minus metrics that fight framework-style APIs. -->
  <rule ref="category/java/design.xml">
    <exclude name="LawOfDemeter"/>
    <exclude name="LoosePackageCoupling"/>
    <exclude name="TooManyMethods"/>
    <exclude name="ExcessiveImports"/>
    <exclude name="DataClass"/>
    <exclude name="ExcessivePublicCount"/>
    <exclude name="CyclomaticComplexity"/>
    <exclude name="CognitiveComplexity"/>
    <exclude name="NPathComplexity"/>
    <exclude name="GodClass"/>
    <exclude name="TooManyFields"/>
    <exclude name="AvoidCatchingGenericException"/>
    <exclude name="SignatureDeclareThrowsException"/>
    <exclude name="AvoidRethrowingException"/>
  </rule>

  <!-- Code style: keep the substantive rules, drop the cosmetic ones.
       MethodArgumentCouldBeFinal stays on as policy. -->
  <rule ref="category/java/codestyle.xml">
    <exclude name="OnlyOneReturn"/>
    <exclude name="AtLeastOneConstructor"/>
    <exclude name="LongVariable"/>
    <exclude name="ShortVariable"/>
    <exclude name="ShortClassName"/>
    <exclude name="ShortMethodName"/>
    <exclude name="CommentDefaultAccessModifier"/>
    <exclude name="LocalVariableCouldBeFinal"/>
    <exclude name="ConfusingTernary"/>
    <exclude name="UselessParentheses"/>
    <exclude name="UnnecessaryLocalBeforeReturn"/>
    <exclude name="UseExplicitTypes"/>
    <exclude name="LinguisticNaming"/>
    <exclude name="FieldNamingConventions"/>
    <exclude name="CallSuperInConstructor"/>
    <exclude name="EmptyMethodInAbstractClassShouldBeAbstract"/>
  </rule>
</ruleset>
```

Test sources usually get a smaller ruleset of their own (`config/pmd/pmd-ruleset-test.xml`, wired through `tasks.named('pmdTest') { ruleSetFiles = files('config/pmd/pmd-ruleset-test.xml'); ruleSets = [] }`). It holds only the policy rules, such as `category/java/codestyle.xml/MethodArgumentCouldBeFinal` and `category/java/bestpractices.xml/AvoidReassigningParameters`, because behaviour-style tests trip many of the main rules on purpose.

## 4. Audit and fix

All modes:
1. Run the gates that exist, with `--continue` so one failing gate doesn't hide the others: for example `./gradlew spotlessCheck pmdMain pmdTest checkstyleMain --continue`, dropping any task that isn't in this build.
2. Group the findings by rule, with counts and the worst files.
3. Note any recurring review comment that an automated rule could catch, so it never comes up again.

`fix` mode only (an `audit` changes nothing):
4. **Formatting:** run the formatter (`./gradlew spotlessApply`); never hand-format.
5. **Lint findings:** fix them in the code. Suppress only a genuine false positive, naming the rule and the reason. Never weaken the ruleset to get green unless the user asked for a policy change.
6. Re-run the specific gate first, then `./gradlew check --continue`.

## Output

- `audit`: findings by rule (count, example `file:line`, fix), the gates missing from the build, and the order to fix things in.
- `fix`: the changes made, and the gate output before and after.
- `setup`: the config files, the build wiring, a one-line reason for every exclusion, and the backlog size from the report-mode run.

## Avoid these errors

- **PMD 6 versus 7.** A PMD 6 ruleset can fail under PMD 7 with "Unable to find referenced rule". Keep the ruleset in step with `toolVersion`.
- **Checkstyle versions must match.** The config has to come from the same release as the tool, or the task fails before it checks anything. Set `maxWarnings = 0`, or Google's config never fails anything.
- **One formatting style.** Pair google-java-format with Google's Checkstyle config; both use 2-space indentation. Don't combine it with a Sun-style config.
- **Line endings.** With `lineEndings 'UNIX'`, `spotlessCheck` fails on CRLF checkouts, which is the Git for Windows default (`core.autocrlf=true`). Commit a `.gitattributes` containing `* text=auto eol=lf` so every checkout gets LF, or leave Spotless on its default, which follows git's settings.
- **Old formatter on a new JDK.** If google-java-format fails with an `IllegalAccessError`, upgrade the Spotless plugin rather than adding JVM flags.
- **Plugin ids.** Core plugins (`pmd`, `checkstyle`) take no version. Community plugins need the exact id and a version: `com.diffplug.spotless`, `com.github.spotbugs`, `net.ltgt.errorprone`.
