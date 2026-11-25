---
description: Set up, harden or fix a Gradle Java build with a toolchain, one place for versions, the test stack and quality gates
argument-hint: "[module | multi-module root | harden | fix]"
allowed-tools: Read, Grep, Glob, Write, Edit, Bash(./gradlew:*), Bash(gradle:*)
disable-model-invocation: true
---

# Gradle build

**Target:** $ARGUMENTS

Make the build reproducible and gated: a pinned Java toolchain, one place for dependency versions, the JUnit 5 test stack, quality gates wired into `check`, and publishing when other modules consume this one. For an existing build, make the smallest change that fixes or hardens it. With no target, work on the build in the current directory: set it up if there is none, otherwise harden it. With `fix`, follow section 4.

## 1. Preflight (if a check fails, say so and stop, or narrow the job)

1. **Find the build and its DSL.** `build.gradle` and `settings.gradle` mean Groovy DSL; `.kts` files mean Kotlin DSL. Write every snippet in that DSL. If there's only a `pom.xml`, say this command targets Gradle and ask before converting anything.
2. **Read the repo's own rules first:** `CLAUDE.md`, `README`, `CONTRIBUTING`, and the comments in the build files. Where they differ from this command, the repo wins.
3. **Inventory before you edit; don't guess:**
   - the Gradle version (`gradle/wrapper/gradle-wrapper.properties`);
   - the Java version (toolchain, or `sourceCompatibility`);
   - the applied plugins and their versions;
   - where versions live: `gradle/libs.versions.toml`, an `ext {}` block, or literals;
   - which gates are wired, and their thresholds;
   - how sibling modules are consumed: a multi-project `settings.gradle`, `includeBuild`, or `mavenLocal()`.
4. **Check there's real work.** If a setting already exists, report it and propose only the delta. Don't rewrite a working build, and don't swap one convention for another (an `ext {}` block for a catalog, say) unless asked.
5. **Check you can verify.** You need the wrapper, a JDK the toolchain accepts, and network access to the Gradle Plugin Portal and Maven Central, or a warm Gradle cache. If any is missing, make the change, label it unverified, and give the exact command to run.

## 2. A new build

For a new module, match the Gradle and Java versions the sibling repos use (read their wrapper properties and toolchain). These files were checked on Gradle 8.14 with Java 21:

`settings.gradle`
```groovy
rootProject.name = 'example-connector'
```

`gradle/libs.versions.toml`
```toml
[versions]
junit = "5.14.4"
mockito = "5.23.0"

[libraries]
junit-bom = { module = "org.junit:junit-bom", version.ref = "junit" }
junit-jupiter = { module = "org.junit.jupiter:junit-jupiter" }
junit-platform-launcher = { module = "org.junit.platform:junit-platform-launcher" }
mockito-core = { module = "org.mockito:mockito-core", version.ref = "mockito" }
mockito-junit-jupiter = { module = "org.mockito:mockito-junit-jupiter", version.ref = "mockito" }
```

`build.gradle`
```groovy
plugins {
  id 'java-library'
  id 'jacoco'
}

group = 'com.example'
version = '0.1.0'

repositories {
  mavenCentral()
}

java {
  toolchain { languageVersion = JavaLanguageVersion.of(21) }
}

tasks.named('compileJava') {
  options.encoding = 'UTF-8'
  options.compilerArgs += ['-parameters']
}

dependencies {
  testImplementation platform(libs.junit.bom)
  testImplementation libs.junit.jupiter
  testRuntimeOnly libs.junit.platform.launcher
}

tasks.named('test') {
  useJUnitPlatform()     // without it, Gradle 8 runs zero JUnit 5 tests and reports success; Gradle 9 fails
  finalizedBy tasks.named('jacocoTestReport')
  testLogging { events 'passed', 'skipped', 'failed'; exceptionFormat 'full' }
}
```

- Add the wrapper with `gradle wrapper --gradle-version <the version the sibling repos use>`. That needs a local Gradle install; without one, say so and ask the user to run it.
- Add a `.gitignore` with `.gradle/` and `build/`, and a `.gitattributes` with `* text=auto eol=lf` if the team works on both Windows and macOS or Linux.
- Mockito is in the catalog but not in `dependencies`; add it only if the tests will use it (with `mockito-junit-jupiter` for `MockitoExtension`).

## 3. Harden an existing build

1. **Toolchain.** Pin the Java version the repo already targets:
   ```groovy
   java {
     toolchain { languageVersion = JavaLanguageVersion.of(21) }
     // for a library other modules compile against:
     // withSourcesJar()
     // withJavadocJar()   // puts javadoc into assemble, so Javadoc errors start failing the build
   }
   ```
2. **Dependencies.** Keep one source of versions, in the form the repo already uses.
   - Use `api` only for types that appear in your public signatures. It needs the `java-library` plugin; under plain `java` the build fails with "Could not find method api()".
   - Keep the JUnit launcher on `testRuntimeOnly`. Gradle 8 warns without it, and Gradle 9 fails.
   - Libraries listed without a version (the JUnit ones above) take it from the BOM.
3. **Test task:** `useJUnitPlatform()` and readable logging, as in the new-build file above. If tests write files, point them under `build/` with a `systemProperty`, so `clean` removes them.
4. **Quality gates.** Add only the gates the user asked for, or the ones the repo already standardises on. Each gate needs its plugin, its config file, a pinned version, and wiring into `check`; see the reference below. Keep `ignoreFailures = false`.
5. **Multi-module and multi-repo builds.**
   - **One repo, several modules:** a root `settings.gradle` with `include`, shared configuration in a convention plugin (`buildSrc` or `build-logic`) rather than copy-paste, and an acyclic dependency graph.
   - **Separate builds that consume each other through `mavenLocal()`,** which is common in connector families (a core contract, one source implementation per connector, one host adapter per connector): apply `maven-publish` and publish upstream first, or downstream fails with "Could not find group:artifact:version". To share a fake or fixture with downstream tests, apply `java-test-fixtures` and consume it with `testImplementation(testFixtures("group:artifact:version"))`. If code reads its own version at runtime, stamp `Implementation-Title` and `Implementation-Version` into the jar manifest.
6. **Reproducibility.** Commit the wrapper (`gradlew`, `gradlew.bat`, `gradle/wrapper/*`). No machine-specific paths. Keep credential files out of jars with `processResources { exclude 'secrets.properties' }`, using your file's name. If the team syncs through a cloud drive, keep OS junk out of archives:
   ```groovy
   tasks.withType(Jar).configureEach { exclude '**/desktop.ini', '**/.DS_Store' }
   ```

## 4. Fix a failing build (target `fix`)

1. Re-run the failing task alone with `--stacktrace` (and `--info` if the cause isn't clear), scoped to the module in a multi-project build.
2. Read the first real error, not the last line. Match it against "Avoid these errors" below: a missing plugin or task, a `tasks.named` on an absent task, a stale `mavenLocal()` artifact, a blocked network, a JDK the toolchain doesn't accept.
3. Make the smallest change that fixes the cause, and re-run the task, then `./gradlew build`.
4. If the failure is a gate doing its job (a lint finding, a coverage floor, a failing test), fix the code, not the gate, unless the user asks for a policy change.

## 5. Gate reference (Groovy DSL)

Versions marked *known-good* come from working Java 21 / Gradle 8.10 connector builds (October 2026). For anything else, take the current version from the plugin's own page, check it supports your Gradle and Java versions, and run `./gradlew help` once after adding it. Never leave a placeholder version in a build file; if you can't look one up, don't add the gate, and tell the user exactly what to add.

**Spotless (formatting).** Plugin `id 'com.diffplug.spotless' version '8.9.0'` (known-good).
```groovy
spotless {
  lineEndings 'UNIX'            // expects LF everywhere; commit a .gitattributes of "* text=auto eol=lf" with it
  java {
    target 'src/**/*.java'
    googleJavaFormat()
    removeUnusedImports()
    trimTrailingWhitespace()
    endWithNewline()
    // only if the file exists:
    // licenseHeaderFile(rootProject.file('config/spotless/license-header.txt'))
  }
}
```

**PMD.** Core plugin `id 'pmd'`, which takes no plugin version. Known-good tool version `7.26.0`.
```groovy
pmd {
  toolVersion = '7.26.0'
  consoleOutput = true
  ignoreFailures = false
  ruleSetFiles = files('config/pmd/pmd-ruleset.xml')
  ruleSets = []                 // required with ruleSetFiles, or Gradle's default ruleset runs as well
}
```
PMD 7 renamed and removed rules from PMD 6; a PMD 6 ruleset can fail with "Unable to find referenced rule". For a starter PMD 7 ruleset, suggest the `java-standards` command with `setup`.

**JaCoCo.** Core plugin `id 'jacoco'`. Known-good `0.8.15`; Java 21 needs 0.8.11 or later.
```groovy
jacoco { toolVersion = '0.8.15' }
tasks.named('jacocoTestReport') {
  dependsOn tasks.named('test')
  reports { xml.required = true; html.required = true }
}
tasks.named('jacocoTestCoverageVerification') {
  dependsOn tasks.named('test')
  violationRules { rule { limit { counter = 'INSTRUCTION'; minimum = 0.75 } } }
}
tasks.named('check') { dependsOn tasks.named('jacocoTestCoverageVerification') }
```
Read the current coverage before you set `minimum`; a floor above it fails `check` straight away.

**Checkstyle.** Core plugin `id 'checkstyle'`. Set `toolVersion` to a current release that supports your Java syntax, `configFile = file('config/checkstyle/checkstyle.xml')`, `maxWarnings = 0` (Google's config reports at WARNING, so without it nothing fails) and `ignoreFailures = false`. Copy `google_checks.xml` from the same release as `toolVersion`.

**SpotBugs.** Plugin `id 'com.github.spotbugs'`; there is no plugin called `spotbugs`. Keep the configuration to `spotbugs { ignoreFailures = false }`, because the types of `effort` and `reportLevel` changed between plugin 5.x and 6.x.

**Error Prone.** Plugin `id 'net.ltgt.errorprone'`, plus the dependency `errorprone 'com.google.errorprone:error_prone_core:<current version>'`; without it, compilation fails. Turn it on after a clean baseline.

**Optional: format, then lint, on every compile.** Only if the team wants it: it rewrites sources during the build, and every task that compiles also runs PMD.
```groovy
tasks.named('compileJava') { dependsOn 'spotlessApply'; finalizedBy 'pmdMain' }
tasks.named('pmdMain') { mustRunAfter 'spotlessApply', 'compileJava' }
```

## 6. Kotlin DSL equivalents

```kotlin
plugins { `java-library`; jacoco; pmd }
java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }
dependencies {
  testImplementation(platform("org.junit:junit-bom:5.14.4"))
  testImplementation("org.junit.jupiter:junit-jupiter")
  testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}
tasks.named<Test>("test") { useJUnitPlatform(); finalizedBy(tasks.named("jacocoTestReport")) }
pmd {
  toolVersion = "7.26.0"
  isConsoleOutput = true
  isIgnoreFailures = false
  ruleSetFiles = files("config/pmd/pmd-ruleset.xml")
  ruleSets = listOf()
}
tasks.named<JacocoCoverageVerification>("jacocoTestCoverageVerification") {
  violationRules { rule { limit { counter = "INSTRUCTION"; minimum = "0.75".toBigDecimal() } } }
}
```

## Avoid these errors

- **Plugin ids.** Core plugins (`java`, `java-library`, `application`, `jacoco`, `pmd`, `checkstyle`, `maven-publish`, `java-test-fixtures`) take no version. Community plugins need the exact id and a version: `com.diffplug.spotless`, `com.github.spotbugs`, `net.ltgt.errorprone`, `info.solidsoft.pitest`, `org.owasp.dependencycheck`.
- **Task names.** `tasks.named('x')` fails with "Task with name 'x' not found" when the plugin that creates `x` isn't applied. For optional wiring, use `tasks.matching { it.name == 'x' }.configureEach { ... }`, which does nothing when `x` is absent. Check a task exists (`./gradlew tasks --all`) before suggesting a command.
- **New gates on old code** usually fail on day one. Run the gate once in report mode (`ignoreFailures = true`), fix or baseline the findings, then enforce.
- **Resolve configurations late.** Resolving one while the script is evaluated (for example `configurations.runtimeClasspath.files` inside a `manifest {}` block) breaks `./gradlew tasks` offline and the configuration cache. Do it inside `doFirst {}` or through a lazy provider.
- **mavenLocal chains.** A downstream "Could not find ..." usually means the upstream module wasn't republished after its last change.
- **Line endings.** `lineEndings 'UNIX'` makes `spotlessCheck` fail on CRLF checkouts, which is the Git for Windows default. Keep it only together with a committed `.gitattributes` of `* text=auto eol=lf`; otherwise leave Spotless on its default, which follows git's settings.
- **Blocked network.** If the plugin portal or Maven Central is blocked, the build fails before any code runs. Report the blocked host; don't edit the build to work around a network policy.

## Verify

- `./gradlew help` succeeds after any plugin change; it's a fast configuration check.
- `./gradlew build --dry-run` lists the gate tasks in the graph.
- `./gradlew build` passes, and introducing a style, lint or coverage violation fails the right task.
