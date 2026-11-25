---
description: Write or fix Javadoc and package-info for a class, package or module, or build the searchable HTML API site
argument-hint: "[Class | package | module | site]"
allowed-tools: Read, Grep, Glob, Write, Edit, Bash(./gradlew:*), Bash(javadoc:*)
disable-model-invocation: true
---

# Javadoc

**Target:** $ARGUMENTS

Document the public API accurately, and make `javadoc` build cleanly. If the target is `site`, generate the HTML API site (section 4). If no target is given, list the public types in the current module with missing or broken Javadoc, show the top gaps, and ask which to fix before editing anything.

## 1. Preflight

1. **Find the target and its public surface.** If the class or package doesn't exist, say so and list near matches. If everything in scope is private, say the effort belongs on the public API, and name it.
2. **Check the code is stable.** If the API is mid-rewrite, say that docs written now will go stale, and ask before going ahead.
3. **Document only what's true.** Read the signatures and the behaviour first. If behaviour is unclear, ask or flag it; never write a plausible guess.
4. **Learn the house style.** Read two or three of the best-documented classes in the repo and match them. Check the build's doclint setting (`Xdoclint` in the `javadoc` task's options).
5. **Leave generated documents alone.** Files marked as generated, or written by a script (HTML walkthroughs, reports, `docs/apidocs/`), are regenerated, never hand-edited.
6. **Check `javadoc` can run here.** If it can't, deliver the changes and say they're unverified.

## 2. Write the Javadoc

- **The first sentence is a specific summary,** because it becomes the index entry: "Returns an immutable view of the buffered rows", not "Gets the rows".
- Follow it with `<p>` paragraphs on behaviour, side effects, thread-safety and nullability.
- Add every `@param`, `@return` and `@throws`, including unchecked exceptions that are part of the contract.
- Use `{@link}` and `{@code}` for identifiers and code. Add `@since` to new API once the project uses it, and `@deprecated` naming the replacement, together with the `@Deprecated` annotation.
- Document the contract, not the code. `@Override` methods inherit their docs unless the behaviour differs.
- **Comments in the code** explain why: intent, trade-offs, gotchas, measured facts, dated decisions. Delete stale or obvious comments. Name the method you mean; don't cite line numbers, because they rot.

## 3. package-info.java (one per package)

```java
/**
 * <b>One line on what this package is for.</b>
 *
 * <h2>Why it exists</h2>
 *
 * <p>The responsibility it owns, and what it deliberately does not do.
 *
 * <ul>
 *   <li>{@link com.example.demo.Greeter} — what this key type does.
 * </ul>
 */
package com.example.demo;
```

If the repo keeps `README`, `CONTRIBUTING` or `CLAUDE.md` files and the documented behaviour changed, update them in their existing format, and make sure every command you put in a doc actually runs.

## 4. The HTML API site (target `site`, or on request)

From JDK 9 on, `javadoc` writes a searchable site with no extra plugin: `index.html`, a working search box, `allclasses-index.html`, `index-all.html` and `overview-tree.html`. This Groovy DSL was run on Gradle 8.14 and Java 21:

```groovy
javadoc {
  options {
    encoding = 'UTF-8'
    windowTitle = "${project.name} ${project.version} API"
    docTitle = "${project.name} ${project.version} API"
  }
}
tasks.register('apidocs', Sync) {   // publishes the site next to the code, as docs/apidocs/
  from tasks.named('javadoc')
  into layout.projectDirectory.dir('docs/apidocs')
}
```

- **Doclint.** Keep the repo's setting. On JDK 21, doclint already runs every check by default; `-Xdoclint:all,-missing` loosens it (it drops the missing-comment warnings) rather than tightening it. To make warnings fail the build, add `addBooleanOption('Werror', true)` to the options, but only once `./gradlew javadoc` runs without warnings. Treat an existing `-Xdoclint:none` as debt: it hides real errors.
- **Links to the JDK API.** `links 'https://docs.oracle.com/en/java/javase/21/docs/api/'` fetches over the network at build time. Leave it out on offline or locked-down machines.
- **Without Gradle:** `javadoc -d docs/apidocs -sourcepath src/main/java -subpackages com.example -encoding UTF-8 -windowtitle "name version API"`, with the repo's real root package. If the code imports anything outside the JDK, add `-classpath` with the dependency jars, or javadoc fails with "package does not exist".
- **Publishing a library:** `java { withJavadocJar() }` puts `javadoc` into `assemble`, so from then on Javadoc errors fail the build.

## 5. Avoid these errors (doclint is on by default)

Each of these fails the `javadoc` task on JDK 21, even without any `-Xdoclint` flag:

| Mistake | Fix |
|---|---|
| Self-closing tags such as `<p/>` or `<br/>` | `<p>` on its own line; no `<br>` |
| A raw `<`, `>` or `&` in text, such as `List<String>` | `{@code List<String>}`, or `&lt;` `&gt;` `&amp;` |
| A heading out of sequence | Class and package docs start at `<h2>`; method and field docs at `<h4>`; an `overview.html` file at `<h1>` |
| `{@link Missing}` or `@see Missing` for a type javadoc can't find | Use the fully qualified name, or a type on the javadoc classpath |
| `@param` with a name that isn't a parameter | Match the parameter names exactly |

- Missing comments are only warnings, unless the build adds `-Werror`. `-Xdoclint:none` turns the errors above into warnings: it hides problems rather than fixing them.
- `package-info.java` follows the same rules.
- Run only tasks that exist. `checkstyleMain` exists only when Checkstyle is applied.

## 6. Verify

- `./gradlew javadoc` builds with no errors, and so does `./gradlew checkstyleMain` if Checkstyle is applied. In a multi-project build, scope them, as in `./gradlew :module:javadoc`.
- Spot-check three entries against the code's actual behaviour.
- For the site, open `docs/apidocs/index.html` and check the title, that a class name and a method name both resolve in the search box, and that `allclasses-index.html` lists every public type.
