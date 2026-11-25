---
description: Write JUnit tests for a Java class, method or bug with the test stack the repo already declares
argument-hint: "[ClassName | path/to/File.java | Class#method | bug description]"
allowed-tools: Read, Grep, Glob, Write, Edit, Bash(./gradlew:*), Bash(gradle:*), Bash(./mvnw:*), Bash(mvn:*), Bash(git diff:*), Bash(git status:*)
disable-model-invocation: true
---

# Write Java tests

**Target:** $ARGUMENTS

Write tests that compile, run and catch regressions. Cover the happy path, every branch and guard, edge cases and error paths.

## 1. Resolve the target

- **A class, file or method:** test that unit.
- **A bug description:** write a regression test that fails on the current code first. Fix the code only if the user asks. Name the bug and the date in the test's comment or display name.
- **No target:** take the production classes changed in the working tree that have no matching tests: `git diff --name-only HEAD`, plus new files git doesn't track yet (`git status --porcelain` lists them as `??`). If there are none, ask which class to test, and stop.
- If the class can't be found, say so and list near matches.
- Records, DTOs and plain getters don't need dedicated tests. Say so, and test the behaviour that uses them.

## 2. Ground rules

- **Read the repo first:** the build files and the repo's own rules (`CLAUDE.md`, `README`, `CONTRIBUTING`). Where they differ from this command, the repo wins.
- **Use only what's declared.** Detect the test stack from the build file, not from memory (table below). Don't add a dependency unless the user agrees. If a library you'd normally use is missing, write the test with what's there.
- **Maven repos:** run tests with `./mvnw test -Dtest=ClassName` (or `mvn`), and read dependencies from `pom.xml`.
- **Label what you couldn't run.** If tests can't run here, say they're unverified and give the exact command.

## 3. Detect the test stack

| To use | The test classpath must have |
|---|---|
| JUnit 5 (`org.junit.jupiter`) | `junit-jupiter`, and `useJUnitPlatform()` on the test task. Without it, Gradle 8 runs zero tests and still reports success (with only a deprecation warning), and Gradle 9 fails the build. If the repo uses JUnit 4 (`junit:junit`), write JUnit 4. |
| `@ParameterizedTest` | `junit-jupiter-params`, which the `junit-jupiter` aggregate includes |
| AssertJ `assertThat` | `org.assertj:assertj-core`. Otherwise use `org.junit.jupiter.api.Assertions`. |
| Mockito `@Mock`, `given(...)` | `org.mockito:mockito-core` |
| `@ExtendWith(MockitoExtension.class)` | `org.mockito:mockito-junit-jupiter` as well. With `mockito-core` alone, the import doesn't compile. |
| Testcontainers | its JUnit 5 module, and a running Docker daemon |

Then:
- **Reuse fixtures.** Look in `src/testFixtures`, in `testkit` or `testing` packages, and for classes named `Fake*`, `InMemory*` or `Stub*`. Connector repos often ship a fake of the source system; use it instead of mocking HTTP.
- **Check testability.** Hidden clocks, `new` of collaborators and static I/O need a seam first, such as constructor injection or an interface. Propose the smallest seam rather than writing a contorted test.
- **Extend, don't duplicate.** If tests already cover the behaviour, extend them or report what's covered.
- **Match local conventions:** test naming, assertion style (qualified `Assertions.assertEquals(...)` or static imports), `@DisplayName`, Javadoc on test classes, and `final` parameters if the repo's PMD rules apply to tests. In that case write `@TempDir final Path dir` too.

## 4. Write the tests

1. **List the cases first,** one per behaviour, branch, boundary or error.
2. **One behaviour per test,** named for its intent: `transferMovesFundsAndNotifiesRecipient` or `transfer_insufficientFunds_throws`, whichever style the repo uses. Structure each test as given, when, then.
3. **Prefer fakes** for collaborators the repo owns: in-memory repositories, recording notifiers, the fake source. Fakes let you assert state, and they survive refactors. Use Mockito where the repo already does, or for collaborators that are hard to fake.
4. **Assert behaviour, not implementation.** For error paths, check the exception type and its message. Give every assertion a message that says what must hold.
5. **Tables of cases** use `@ParameterizedTest` with `@CsvSource`, `@MethodSource`, `@ValueSource` or `@NullAndEmptySource`. Share setup in `@BeforeEach`.
6. **Files and time:** files go in `@TempDir`, never outside it. Inject a clock; never `Thread.sleep` and hope a race resolves.
7. **Integration tests** run the real adapter against the real dependency, with URLs and credentials resolved at runtime, never hard-coded. Guard Docker-based tests with `@Testcontainers(disabledWithoutDocker = true)`, keep the container in a `static` field, and take module and class names from the docs of the declared library version.

## 5. Templates

These templates were compiled and run on Java 21 with JUnit 5, AssertJ and Mockito, against a small example: `AccountService(AccountRepository, NotificationGateway)`, where `Account` is `record Account(String id, long balance)`. A planted bug (notifying the payer instead of the payee) made A and B fail as intended. Rename the types to your own, and keep the imports.

### A. JUnit 5 only, with fakes (works in any JUnit 5 setup)

```java
package com.example.accounts;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

/** Pins that a transfer moves funds and notifies, and that an overdraft changes nothing. */
class AccountServiceFakeTest {

  /** In-memory repository: a fake, not a mock, so the test asserts state rather than calls. */
  private static final class InMemoryAccounts implements AccountRepository {
    private final Map<String, Account> rows = new HashMap<>();
    private int saves;

    InMemoryAccounts with(final Account account) {
      rows.put(account.id(), account);
      return this;
    }

    @Override
    public Optional<Account> findById(final String id) {
      return Optional.ofNullable(rows.get(id));
    }

    @Override
    public void save(final Account account) {
      saves++;
      rows.put(account.id(), account);
    }
  }

  /** Records every notification instead of sending it. */
  private static final class RecordingNotifier implements NotificationGateway {
    private final List<String> recipients = new ArrayList<>();

    @Override
    public void notifyCustomer(final String customerId, final String message) {
      recipients.add(customerId);
    }
  }

  private InMemoryAccounts accounts;
  private RecordingNotifier notifier;
  private AccountService service;

  @BeforeEach
  void setUp() {
    accounts = new InMemoryAccounts().with(new Account("a", 100)).with(new Account("b", 0));
    notifier = new RecordingNotifier();
    service = new AccountService(accounts, notifier);
  }

  @Test
  @DisplayName("a transfer moves the funds and notifies the recipient")
  void transferMovesFundsAndNotifiesRecipient() {
    service.transfer("a", "b", 40);

    Assertions.assertEquals(60, accounts.findById("a").orElseThrow().balance(), "payer debited");
    Assertions.assertEquals(40, accounts.findById("b").orElseThrow().balance(), "payee credited");
    Assertions.assertEquals(List.of("b"), notifier.recipients, "only the payee is notified");
  }

  @Test
  @DisplayName("an overdraft throws and changes nothing")
  void overdraftThrowsAndSavesNothing() {
    IllegalArgumentException thrown =
        Assertions.assertThrows(IllegalArgumentException.class, () -> service.transfer("a", "b", 500));

    Assertions.assertTrue(thrown.getMessage().contains("insufficient"), thrown.getMessage());
    Assertions.assertEquals(0, accounts.saves, "nothing saved");
    Assertions.assertTrue(notifier.recipients.isEmpty(), "nobody notified");
  }
}
```

### B. JUnit 5 + Mockito + AssertJ

Use this only if `mockito-core`, `mockito-junit-jupiter` and `assertj-core` are all declared.

```java
package com.example.accounts;

import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.ArgumentMatchers.argThat;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.BDDMockito.given;
import static org.mockito.BDDMockito.then;
import static org.mockito.Mockito.never;

import java.util.Optional;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class AccountServiceTest {

  @Mock AccountRepository repo;
  @Mock NotificationGateway notify;
  @InjectMocks AccountService service;

  @Test
  void transfer_movesFunds_andNotifiesRecipient() {
    given(repo.findById("a")).willReturn(Optional.of(new Account("a", 100)));
    given(repo.findById("b")).willReturn(Optional.of(new Account("b", 0)));

    service.transfer("a", "b", 40);

    then(repo).should().save(argThat(acc -> acc.id().equals("a") && acc.balance() == 60));
    then(notify).should().notifyCustomer(eq("b"), anyString());
  }

  @Test
  void transfer_insufficientFunds_throws_andSavesNothing() {
    given(repo.findById("a")).willReturn(Optional.of(new Account("a", 10)));
    given(repo.findById("b")).willReturn(Optional.of(new Account("b", 0)));

    assertThatThrownBy(() -> service.transfer("a", "b", 50))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("insufficient");
    then(repo).should(never()).save(any());
  }
}
```

### C. Table-driven

```java
package com.example.accounts;

import java.util.stream.Stream;
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.Arguments;
import org.junit.jupiter.params.provider.CsvSource;
import org.junit.jupiter.params.provider.MethodSource;
import org.junit.jupiter.params.provider.NullAndEmptySource;
import org.junit.jupiter.params.provider.ValueSource;

/** Table-driven examples. */
class ParameterizedExamplesTest {

  @ParameterizedTest(name = "{0} - {1} leaves {2}")
  @CsvSource({"100, 40, 60", "100, 100, 0", "5, 0, 5"})
  void debitLeavesTheRemainder(final long balance, final long amount, final long remainder) {
    Assertions.assertEquals(remainder, balance - amount);
  }

  static Stream<Arguments> accounts() {
    return Stream.of(Arguments.of(new Account("a", 1), 1L), Arguments.of(new Account("b", 0), 0L));
  }

  @ParameterizedTest
  @MethodSource("accounts")
  void balanceIsReadBack(final Account account, final long expected) {
    Assertions.assertEquals(expected, account.balance());
  }

  @ParameterizedTest
  @NullAndEmptySource
  @ValueSource(strings = {" ", "\t"})
  void blankIdsAreBlank(final String id) {
    Assertions.assertTrue(id == null || id.isBlank());
  }
}
```

## 6. Run the tests and prove they bite

1. Run the new class: `./gradlew test --tests 'com.example.accounts.AccountServiceTest'`. In a multi-project build, use `./gradlew :module:test --tests '...'`.
2. Run the module's whole test task.
3. **Sanity-mutate.** Break the code under test on purpose (drop a guard, swap two calls, notify the wrong party), confirm the right test fails, then restore the code exactly and run the tests again.
4. **Report** the tests added, what each one pins, and the result. With parameterized tests, say whether a count means methods or executions.

## In connector repos

- **Drive the fake source.** If the repo has a fake of the source system, teach it the real behaviour first, including the ugly parts: unknown fields silently ignored, headers that misreport totals, audit rows that lag. If the fake can't express a failure, a green test proves nothing.
- **Bound end-to-end runs** that drive the host's own driver: set a record limit, keep the change feed open so a restart test measures something, and assert on the output the host actually receives.
- **Pin every fixed bug** with a regression test that names the bug and the date, and fails before the fix.
- **Leave live runs alone.** Tests against a real instance need credentials and write data, so never run them without the user's go-ahead.

## Avoid these errors

- Copy each template's import block together with the class body.
- `MockitoExtension` uses strict stubs: a stub the test never uses fails it with `UnnecessaryStubbingException`. Remove the stub, or mark it `lenient()` if it's genuinely conditional.
- Mockito releases before 5.x can fail on a new JDK with a Byte Buddy "Java NN is not supported" error. Upgrade Mockito rather than adding experimental flags.
- Don't mix JUnit 4 and JUnit 5 in one class. Running JUnit 4 tests on the JUnit Platform needs the vintage engine.
- Never use `@Disabled` to get green. If a test's premise is wrong, fix the test or the fixture.
- Keep unit tests isolated: no real network, no real clock waits, no `System.exit`, no dependence on test order.
