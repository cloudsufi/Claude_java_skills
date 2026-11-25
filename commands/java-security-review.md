---
description: OWASP security review of Java code or a diff, covering injection, secrets, XML, SSRF, TLS, crypto and dependencies
argument-hint: "[module | endpoints | diff]"
allowed-tools: Read, Grep, Glob, Bash(git diff:*), Bash(git log:*), Bash(git status:*), Bash(git grep:*), Bash(git merge-base:*), Bash(git rev-parse:*), Bash(./gradlew:*)
disable-model-invocation: true
---

# Java security review

**Target:** $ARGUMENTS

List the security issues by risk (Critical, High, Medium, Low). Each one gives the vulnerable `file:line`, the OWASP category, a concrete exploit scenario, a fix, and the automated control that would stop it coming back.

## 1. Resolve the target

- **A module, package or set of endpoints:** review that code.
- **`diff`, or no target:** review the uncommitted changes: `git diff HEAD`, plus new files git doesn't track yet (`git status --porcelain` lists them as `??`). If there are none, review the current branch against the default branch (`git diff origin/main...HEAD`, using the repo's real default branch). If that's empty too, review the module in the current directory as an audit.
- If the target doesn't exist, say so and stop. Never review from memory.

## 2. Preflight

1. **Map the trust boundaries.**
   - Inputs: HTTP parameters, bodies and headers; files; messages; environment variables; config files; responses from external APIs.
   - Sinks: databases and query languages, the shell, the file system, outbound HTTP, deserializers, logs.

   If the code has no trust boundary at all (a pure library with no I/O), say the OWASP surface is minimal and scope the review honestly. Don't manufacture findings.
2. **Read the repo's security conventions first:** how secrets are supplied (environment variables, absolute-path property files, a vault), redaction helpers, "secret" or "sensitive" flags on config fields, and the security rules already in the PMD or SpotBugs config. Recommend what exists before inventing something new.
3. **Don't claim a scan ran if it didn't.** Dependency scanners and SpotBugs need their plugins and network access. If they can't run here, review by reading the code, and list the scan as a recommendation.

## 3. Checklist (OWASP-aligned)

- **Injection.** SQL uses bound parameters, never string concatenation. The same applies to any query language built from input: SOQL, OData `$filter`, GraphQL, or a source's own encoded-query syntax. Escape values or allow-list them by shape (an id, an ISO timestamp, a discovered field name), and check field names against the discovered schema. No shell concatenation. Encode output where HTML is produced.
- **Authentication and access control.** Authorization is enforced server-side on every protected operation. Tokens are refreshed, minimally scoped, and never logged. Passwords are stored with bcrypt or argon2.
- **Secrets and sensitive data.** No secrets in code, logs, exceptions, test fixtures or committed property files. Secrets load from the environment, a vault, or a configured file kept outside the repository (where the repo requires absolute paths for it, a relative path is refused). Secret config values are masked wherever configuration is printed. Credential files stay out of jars (`processResources { exclude ... }`) and out of version control (`.gitignore`).
- **XML and deserialization.** Untrusted XML, including payloads from a source API such as audit records or SOAP, is parsed only with a hardened parser (below). No `ObjectInputStream` on untrusted data, and no Jackson default typing on untrusted input.
- **SSRF and path traversal.** Configured base URLs are validated: `https` and an allowed host, with no blind redirects to other hosts. File paths are canonicalized and checked to stay under the intended base directory.
- **TLS.** Certificate and hostname verification are never disabled: no trust-all `TrustManager`, no `HostnameVerifier` that always returns true.
- **Crypto.** JDK only, strong algorithms, `SecureRandom`. AES-GCM with a fresh 12-byte IV per message. Never ECB, never a reused IV, never home-grown crypto, never MD5 or SHA-1 for security.
- **Data at rest.** If the code persists source data (buffers, caches, spill files), it's encrypted, and the keys are stored apart from the data.
- **Logging and diagnostics.** Headers whose names contain `auth`, `token`, `secret`, `key`, `password`, `cookie` or `credential` are redacted. Response bodies are data, so wire traces stay off by default.
- **Dependencies.** No known-vulnerable versions: run a scan or check the advisories. Keep the dependency surface minimal.
- **Error handling.** Fail closed. Don't leak stack traces or internals to callers.

## 4. Verified fixes (Java 21, JDK only)

**Hardened XML parsing.** With the JDK's default `DocumentBuilderFactory`, an XXE payload read a local file. The hardened builder below refused it ("DOCTYPE is disallowed"), the hardened StAX factory refused the entity, and both still parsed ordinary XML. They use `newDefaultInstance()` and `newDefaultFactory()` so they always get the JDK's own parser: with Xerces on the classpath, `newInstance()` returns Xerces, whose `setAttribute(ACCESS_EXTERNAL_DTD, ...)` throws `IllegalArgumentException`. Both were checked with and without Xerces on the classpath.

```java
import javax.xml.XMLConstants;
import javax.xml.parsers.DocumentBuilder;
import javax.xml.parsers.DocumentBuilderFactory;
import javax.xml.parsers.ParserConfigurationException;
import javax.xml.stream.XMLInputFactory;

static DocumentBuilder hardenedDocumentBuilder() throws ParserConfigurationException {
  DocumentBuilderFactory factory = DocumentBuilderFactory.newDefaultInstance();
  factory.setFeature(XMLConstants.FEATURE_SECURE_PROCESSING, true);
  factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
  factory.setFeature("http://xml.org/sax/features/external-general-entities", false);
  factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
  factory.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
  factory.setXIncludeAware(false);
  factory.setExpandEntityReferences(false);
  factory.setAttribute(XMLConstants.ACCESS_EXTERNAL_DTD, "");
  factory.setAttribute(XMLConstants.ACCESS_EXTERNAL_SCHEMA, "");
  return factory.newDocumentBuilder();
}

static XMLInputFactory hardenedXmlInputFactory() {
  XMLInputFactory factory = XMLInputFactory.newDefaultFactory();
  factory.setProperty(XMLInputFactory.SUPPORT_DTD, false);
  factory.setProperty(XMLInputFactory.IS_SUPPORTING_EXTERNAL_ENTITIES, false);
  return factory;
}
```

**AES-GCM sealing.** Checked to round-trip, to detect a flipped bit (`AEADBadTagException`), and to give different ciphertext for the same plaintext.

```java
import java.nio.ByteBuffer;
import java.security.GeneralSecurityException;
import java.security.SecureRandom;
import javax.crypto.Cipher;
import javax.crypto.SecretKey;
import javax.crypto.spec.GCMParameterSpec;

private static final int IV_BYTES = 12;
private static final int TAG_BITS = 128;
private static final SecureRandom RANDOM = new SecureRandom();

/** Returns IV + ciphertext + tag. A fresh random IV every call: never reuse one under the same key. */
static byte[] seal(final SecretKey key, final byte[] plaintext, final byte[] aad)
    throws GeneralSecurityException {
  byte[] iv = new byte[IV_BYTES];
  RANDOM.nextBytes(iv);
  Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
  cipher.init(Cipher.ENCRYPT_MODE, key, new GCMParameterSpec(TAG_BITS, iv));
  cipher.updateAAD(aad);
  byte[] sealed = cipher.doFinal(plaintext);
  return ByteBuffer.allocate(IV_BYTES + sealed.length).put(iv).put(sealed).array();
}

/** Throws AEADBadTagException if the message or its AAD was altered. */
static byte[] open(final SecretKey key, final byte[] message, final byte[] aad)
    throws GeneralSecurityException {
  Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
  cipher.init(Cipher.DECRYPT_MODE, key, new GCMParameterSpec(TAG_BITS, message, 0, IV_BYTES));
  cipher.updateAAD(aad);
  return cipher.doFinal(message, IV_BYTES, message.length - IV_BYTES);
}
```

## 5. Automated controls (recommend them; add them only on request)

- **Secret scan.** A quick check, tested in bash, that finds credential-like assignments in tracked files and in new files git doesn't track yet. Use gitleaks instead if it's installed.
  ```bash
  git grep --untracked -nEi "(password|secret|token|api[_-]?key)[[:space:]]*[=:][[:space:]]*['\"]?[^'\"[:space:]]{8,}"
  ```
- **PMD:** turn on `category/java/security.xml`. A custom XPath rule can flag credential-named variables initialised with string literals.
- **FindSecBugs:** with the SpotBugs plugin (`com.github.spotbugs`) applied, add `dependencies { spotbugsPlugins 'com.h3xstream.findsecbugs:findsecbugs-plugin:<current version>' }`.
- **OWASP Dependency-Check** (plugin `org.owasp.dependencycheck`): it needs an NVD API key in an environment variable, and its first run downloads the vulnerability database, which is slow. Run it on demand or in CI rather than inside `check`.

## Output

For each issue, most dangerous first: the risk, the OWASP category, `file:line`, the exploit scenario, the fix (with code where useful), and the control that prevents a recurrence. End with the scans that ran (and their results) and the scans that are only recommended.

## Avoid these errors

- **Never paste a real secret into the report.** Mask it: the first four characters followed by `***`.
- **Don't run Dependency-Check without an NVD API key and network access.** Report the gap instead.
- **Suggested fixes must compile and keep the tests green.** Suggested crypto is JDK-only and follows the rules above.
- **Name only plugins the build has, or that the user agreed to add,** with the exact ids above. There is no `spotbugs` plugin id without the `com.github.` prefix.
