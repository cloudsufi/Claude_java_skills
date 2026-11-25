# Java Dev Kit: slash commands for connector teams

Twenty-one slash commands for Java and Gradle work. They work in any connector repository: a core library, a source implementation or a host adapter. They are generic versions of the java-dev-kit commands.

The commands run only when you type them; Claude never starts one on its own. Everything after the command name is its argument, for example `/java-review src/main/java/com/acme/Pager.java`.

## Commands

| Command | What it does | With no argument |
|---|---|---|
| `/java-review` | Reviews a file, package, diff, branch or PR: correctness, concurrency, errors, performance, tests and connector data safety. Findings by severity, with fixes. | Reviews the uncommitted changes, or else the branch |
| `/java-test` | Writes JUnit tests for a class, a method or a bug, with the test stack the repo already has. | Tests the changed classes that have no tests |
| `/java-test-plan` | Plans the testing of a feature or module: layers, risk order, fakes, gates and CI. | Plans for the current module |
| `/java-coverage` | Finds untested risky code with JaCoCo, lists the missing cases, and sets coverage and mutation gates. | Uses the current module |
| `/java-standards` | Audits, fixes or sets up formatting and lint gates (Spotless, PMD, Checkstyle, SpotBugs, Error Prone). Takes `audit`, `fix` or `setup`. | Audits the current module |
| `/javadoc` | Writes or fixes Javadoc and package-info, or builds the HTML API site (`site`). | Lists the documentation gaps and asks |
| `/api-docs` | Documents a library API, an HTTP API as OpenAPI 3, a connector's configuration keys, or the source API it calls. | Detects which kinds apply |
| `/adr` | Writes an Architecture Decision Record in the repo's format and numbering. | Asks for the decision |
| `/c4` | Draws C4 context, container and component diagrams of the real system. | Uses the current repository |
| `/design-doc` | Writes a low-level design: classes, contracts, patterns, SOLID checks and diagrams. | Asks for the feature |
| `/java-plan` | Breaks an epic into stories, tasks, acceptance criteria, estimates, milestones and risks. | Asks for the goal |
| `/uml` | Picks and draws the right UML diagram; add `class`, `sequence`, `usecase`, `component` or `state` to choose. | Suggests candidates and asks |
| `/sequence-diagram` | Traces a real call path and draws it. | Lists entry points and asks |
| `/class-diagram` | Draws the types and relationships of a package or set of classes. | Lists packages and asks |
| `/usecase-diagram` | Draws the actors and goals of a system or feature. | Uses the current repository |
| `/gradle-init` | Sets up, hardens or fixes a Gradle build: toolchain, versions, test stack and gates. | Works on the build in the current folder |
| `/refactor` | Plans a test-backed refactoring; add `apply` to carry it out step by step. | Asks what to refactor |
| `/java-security-review` | OWASP review: injection, secrets, XML, SSRF, TLS, crypto and dependencies. | Reviews the pending changes, or else the module |
| `/java-debug` | Reproduces a defect, finds the root cause, fixes it and adds a regression test. | Asks for the symptom |
| `/generate-commit` | Writes a commit message for the staged changes in the repo's commit style; add `commit` to also commit. | Uses the staged changes, or drafts one for all changes |
| `/generate-pr` | Writes a PR title and description for the current branch against a target branch; add `create` to open it as a draft with the GitHub CLI. | Targets `main` |

Three kit commands have new names, because Claude Code has built-in commands called `/plan`, `/debug` and `/security-review`, and a custom command with the same name would replace the built-in. Here they are `/java-plan`, `/java-debug` and `/java-security-review`.

## Installing

**Claude Code, as a plugin.** This repository is also a plugin marketplace, so Claude Code can install the kit straight from GitHub. In a Claude Code session:

```
/plugin marketplace add tanishqgandhi-rgb/Claude_java_skills
/plugin install java-dev-kit@java-dev-kit
```

Commands from a plugin carry the plugin's name: `/java-dev-kit:java-review`, `/java-dev-kit:java-test`, and so on. To pick up a newer version later, run `/plugin marketplace update java-dev-kit`.

**Claude Code, as plain commands.** If you would rather type `/java-review` without the prefix, copy the command files into a commands folder instead:

```bash
git clone https://github.com/tanishqgandhi-rgb/Claude_java_skills.git

# for you, in every repository:
mkdir -p ~/.claude/commands
cp Claude_java_skills/commands/*.md ~/.claude/commands/

# or for everyone working in one repository (run from that repository, then commit .claude/commands):
mkdir -p .claude/commands
cp /path/to/Claude_java_skills/commands/*.md .claude/commands/
```

Then type `/java-review`, `/java-test` and so on. If Claude Code was already running, start a new session to pick the commands up.

**Claude desktop app (Cowork).** Build `java-dev-kit.plugin` from a clone of this repository (see [Updating the kit](#updating-the-kit)), open it in Cowork, for example by attaching it to a chat, and accept the plugin preview. An organization admin can also publish it to the team's plugin list. Commands carry the plugin's name here too: `/java-dev-kit:java-review`. This version replaces the earlier skills-based java-dev-kit plugin, which had the same name.

## How the commands behave

- **They read the repository first:** the build files, `CLAUDE.md`, `README`, `CONTRIBUTING` and the existing rulesets. The repository's own rules win over the command's defaults.
- **They use only what is declared.** A command doesn't use a library, plugin or Gradle task that isn't in the build. For example, `/java-test` writes plain JUnit assertions if AssertJ isn't on the classpath, and no command runs `checkstyleMain` in a repository without Checkstyle. Maven repositories get the Maven equivalents.
- **They don't add tooling unasked.** New plugins, gates and dependencies are suggestions until you agree.
- **They stop instead of guessing.** If the target doesn't exist, the diff is empty or the input is too vague, the command says so and asks.
- **They label unverified work.** If the build can't run (no JDK, no network, a blocked repository), the command says so and gives the exact command to check it.
- **They leave generated files alone,** and change the source or the generator instead.
- **Permissions.** Each command's `allowed-tools` line pre-approves the tools it needs: reading files, read-only `git`, the Gradle wrapper, and editing files for the commands that write them. Anything else still asks for permission.
- **Model invocation is off.** Each file sets `disable-model-invocation: true`, so the commands run only when typed. Delete that line from a file if you want Claude to be able to start that command by itself.

## What was tested

In October 2026, on Claude Code 2.1.299:

- **Loading:** all 21 commands load from `~/.claude/commands`, from a repository's `.claude/commands`, and as a plugin. Arguments are passed through, and a missing argument leaves the target blank, which every command handles. With `disable-model-invocation`, none of them shows up in Claude's automatic skill list.
- **Git commands:** every `git` and `gh` command that `/generate-commit` and `/generate-pr` rely on was run in a test repository with a remote, including committing a message through `git commit -F-`.
- **Gradle, on Gradle 8.14 and Java 21:** the new-build files, toolchain, version catalog, test task, PMD and Checkstyle wiring, the JaCoCo gate (run against a real coverage report, failing below the floor as intended), both integration-test splits (by tag and by class name), the Javadoc site, the overview page, and making Javadoc warnings fail the build.
- **Coverage script:** the JaCoCo summary script ran against a real JaCoCo XML report.
- **Test templates:** compiled and passing under JUnit 5, AssertJ and Mockito. A deliberately planted bug made the right tests fail.
- **Diagrams:** every Mermaid template parses (Mermaid 11), and every PlantUML template passes `plantuml -checkonly` and renders.
- **Security snippets:** the hardened XML parsers block an XXE payload that the JDK's default parser falls for, with and without Xerces on the classpath; AES-GCM sealing round-trips and detects tampering; the secret scan finds credentials in tracked and untracked files.
- **Independent review:** a separate reviewer read the first 19 commands against these rules, and its findings were fixed and re-tested.
- **OpenAPI template:** validates with `swagger-cli` and Redocly.
- **Starter PMD ruleset:** every rule and exclusion matches a ruleset working on PMD 7.26.

Not run, because the plugin site and Maven Central were unreachable from the test machine: Spotless, SpotBugs, Error Prone and PIT. Their snippets are kept minimal, the Spotless one is copied from working connector builds, and the commands flag them as needing a check on first use.

## Updating the kit

1. Edit the command's file under `commands/`.
2. Bump `version` in `.claude-plugin/plugin.json`.
3. Commit and push, so `/plugin marketplace update java-dev-kit` picks the change up.
4. For Cowork, rebuild the plugin file from inside this folder. It is a zip archive of the folder, without the git data:

```bash
zip -r ../java-dev-kit.plugin . -x '.git/*' '*.DS_Store' '*desktop.ini'
```
