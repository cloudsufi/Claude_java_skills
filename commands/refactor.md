---
description: Plan, and on request apply, a safe test-backed refactoring of Java code that keeps behaviour unchanged
argument-hint: "[class | package | code smell] [apply]"
allowed-tools: Read, Grep, Glob, Write, Edit, Bash(./gradlew:*), Bash(./mvnw:*), Bash(git diff:*), Bash(git status:*), Bash(git log:*)
disable-model-invocation: true
---

# Refactor

**Target:** $ARGUMENTS

Refactoring changes structure, never behaviour, and the tests stay green throughout. By default, produce the plan. Apply it only if the target includes `apply` or the user asks for it afterwards. If no target is given, ask what to refactor, and stop.

## 1. Preflight

1. **Read the target first.** If it doesn't exist, say so and list near matches.
2. **Is it worth the churn?** If the code is already clean, or the smells aren't worth the risk, say so and stop. Refactoring for its own sake is pure risk.
3. **Read the repo's rules:** `CLAUDE.md` invariants, `CONTRIBUTING`, and the PMD or Checkstyle rules. A refactoring must not break a written rule, such as "this method stays `final`" or "no thread pools here".
4. **Check the safety net.** Find the tests that cover the target, and run them. If coverage is thin, the first step of the plan is characterization tests that pin today's behaviour, bugs included. If the design makes those tests impossible, say that's the real problem, and make the first step the smallest seam that allows a test.
5. **Before applying, check the working tree** (`git status`). If there are uncommitted changes, ask the user to commit or stash them first, so each step can be reviewed and undone on its own.

## 2. Diagnose

Name the smells with `file:line`, and the move that fixes each:

| Smell | Typical move |
|---|---|
| Long method | Extract method; replace temp with query |
| Large class, several reasons to change | Extract class; move method |
| Duplicated code | Extract method or class; pull up method |
| Feature envy (a method mostly uses another class's data) | Move method |
| Primitive obsession (ids, money, cursors as raw strings or longs) | Introduce a value type (a record) |
| Long parameter list | Introduce a parameter object, or a builder |
| Deep nesting | Guard clauses; decompose the conditional |
| A switch on type, repeated in several places | Replace the conditional with polymorphism, or a strategy |
| Leaky abstraction, depending on a concrete class | Introduce an interface; depend on the abstraction |
| Mutable shared state | Make it immutable, or confine it to one owner |

Check the SOLID principles as you go: single responsibility, open/closed, Liskov substitution, interface segregation, dependency inversion.

## 3. Plan

- An ordered list of small, behaviour-preserving steps, each one independently shippable, with the tests that guard it.
- For each step: what moves where, the public API impact (none, or a deprecation path), and the risk.
- Keep public API changes out of the plan unless asked. If one is needed, deprecate first, naming the replacement.

## 4. Apply (only when asked)

1. One step at a time. After each step, compile and run the tests (`./gradlew test`, scoped to the module in a multi-project build), plus the gates the build has (`./gradlew check`).
2. If a test fails, the step changed behaviour: undo it, or fix the step. Never change a test's expectation to make a refactoring pass, except a characterization test you wrote for a behaviour the user agreed to change.
3. Use automated rename and move where possible, and update every reference, including Javadoc, `CLAUDE.md` and README mentions.
4. Show the diff for each step, and summarize at the end: steps done, tests run, results.

## In connector repos

- Keep the checkpoint ordering, cursor handling and paging logic exactly as they are, unless the refactoring is about them. Those bugs exit 0 and look healthy.
- Don't merge per-pipeline state into shared state for "simplicity"; isolation per pipeline is usually deliberate.
- Keep mechanism in the core and policy in the consumer; a refactoring must not move a dependency outward or between sibling implementations.

## Avoid these errors

- Don't mix a refactoring with a behaviour change in the same step.
- Don't rename or move public types used by other repos (published jars, `mavenLocal()` consumers) without a deprecation path.
- If the build can't run here, say that the steps are unverified and give the exact commands to run after each one.
