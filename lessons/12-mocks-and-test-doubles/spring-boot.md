# 12 — Mocks and test doubles · Spring Boot

[← Concepts](README.md) · Starting point: end of lesson 11 · Estimated time in class: 2 h · Independent work tasks: [README](README.md#independent-work-2-h)

---

## What we build today

- `budget/BudgetMonitorTest.java` — a plain **Mockito** unit test of the budget warning (R9): the
  repository and `ProjectService` **stubbed**, the `Notifier` **mocked**, the `Clock` **fixed**. Sent at 80 %, not at
  79 %, not when already sent, not when another request claimed it first, not for hourly projects; the
  claim is released when sending fails. An `ArgumentCaptor` checks the subject.
- `notification/InMemoryNotifier.java` (test sources) — a hand-written **fake**, used in one test and
  compared with the mock version.
- `timeentry/TimeEntryServiceTest.java` — the service publishes `TimeEntryLoggedEvent` (checked with a
  captor), rejects archived projects and entries in the future, and publishes nothing when it rejects.
- `client/ClientControllerTest.java` — one `@WebMvcTest` with `MockMvcTester` and `@MockitoBean`.
- `ProjectTestData` from lesson 11 gets three more builder methods.

**Final result:** `./mvnw test` is green with 11 new tests (more after the independent work). None of them needs a database or a
mail server; the unit tests run in milliseconds, the web slice in about a second.

### The code we test

These are the classes from lessons 07–11 that today's tests use. If your names or constructor order
differ, adapt the tests — the constructor call is in one `setUp()` method per class.

| Class | What the tests rely on |
|---|---|
| `notification.Notifier` | `void notify(String subject, String message)` |
| `budget.BudgetMonitor` | lesson 10 version plus `reachedThreshold()` from lesson 11 — shown in full in step 2 |
| `budget.TimeEntryLoggedEvent` | `record TimeEntryLoggedEvent(Long projectId)` |
| `timeentry.TimeEntryService` | constructor `(TimeEntryRepository, ProjectRepository, TagRepository, Clock, ApplicationEventPublisher)` (lessons 07 and 09); `TimeEntry log(LogTimeEntryCommand)` |
| `project.ProjectService` | `long billableMinutes(Project)` — the shared minutes method from lesson 10 |
| `timeentry.LogTimeEntryCommand` | `record (long projectId, LocalDateTime startedAt, LocalDateTime endedAt, String description, boolean billable, Set<Long> tagIds)` |
| Exceptions | `project.ProjectArchivedException`, `timeentry.TimeEntryInFutureException`, `common.NotFoundException` for an unknown project id (lesson 07) |
| `client.ClientController` | depends only on `ClientService`; POST `/clients` binds `@ModelAttribute("form") ClientForm`, shows `clients/create` on errors, redirects to `/clients/{id}` |
| `project.Project` | `setBudgetWarningSentAt(LocalDateTime)`, `getBudgetWarningSentAt()`, `archive(LocalDateTime)` |
| `project.ProjectRepository` | `int claimBudgetWarning(long id, LocalDateTime now)`, `int releaseBudgetWarning(long id)` (lesson 09) |

---

## Step 1 — What you already have for mocking

**Why:** Mockito and the web test support are already on the classpath. Check before adding anything.

Open `pom.xml`:

```xml
<!-- pom.xml (excerpt — already there from Initializr) -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-test</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-webmvc-test</artifactId>
  <scope>test</scope>
</dependency>
```

**What happens:**

1. `spring-boot-starter-test` contains **Mockito** (`mockito-core` and `mockito-junit-jupiter`, the JUnit
   extension). That is all a plain unit test with mocks needs.
2. `spring-boot-starter-webmvc-test` brings the `spring-boot-webmvc-test` module. **Spring Boot 4 moved
   the test slices into separate modules and packages**: `@WebMvcTest` is now
   `org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest`. Tutorials for Boot 3 import it from
   `org.springframework.boot.test.autoconfigure.web.servlet` — that package no longer exists.
3. If `spring-boot-starter-webmvc-test` is missing (for example because the project was generated with
   an older Initializr), add the dependency above. The version comes from the Spring Boot parent.

> **Version note.** `@MockBean` and `@SpyBean` were deprecated in Spring Boot 3.4 and **removed in
> Spring Boot 4**. Use `@MockitoBean` and `@MockitoSpyBean` from
> `org.springframework.test.context.bean.override.mockito`. AI assistants still suggest `@MockBean` very
> often.

**Optional: silence the Mockito agent warning.** On Java 21 and newer, the first test with a mock may
print *"Mockito is currently self-attaching to enable the inline-mock-maker. This will no longer work in
future releases of the JDK."* It is a warning, not an error. To follow Mockito's recommendation, load
Mockito as a Java agent. Add to `<build><plugins>` (versions are managed by the Spring Boot parent):

```xml
<!-- pom.xml — inside <build><plugins> -->
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-dependency-plugin</artifactId>
  <executions>
    <execution>
      <goals>
        <goal>properties</goal>
      </goals>
    </execution>
  </executions>
</plugin>
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-surefire-plugin</artifactId>
  <configuration>
    <argLine>@{argLine} -javaagent:${org.mockito:mockito-core:jar}</argLine>
  </configuration>
</plugin>
```

`properties` sets a Maven property with the path of every dependency jar, so
`${org.mockito:mockito-core:jar}` is the path of the Mockito jar. `@{argLine}` keeps the JaCoCo agent from
lesson 11 (JaCoCo's `prepare-agent` sets `argLine`; without JaCoCo, remove `@{argLine}`).

---

## Step 2 — The class under test: `BudgetMonitor`

**Why:** a Mockito test stubs the exact methods the class calls, so we need to see what `BudgetMonitor`
looks like after lessons 09–11. This is the lesson 10 version with the `reachedThreshold()` method from
lesson 11's Task 2. Compare it with yours; if yours differs in details, adapt the stubs in step 4.

```java
// src/main/java/ee/ta25/billable/budget/BudgetMonitor.java
package ee.ta25.billable.budget;

import ee.ta25.billable.billing.HourlyBilling;
import ee.ta25.billable.common.Money;
import ee.ta25.billable.notification.Notifier;
import ee.ta25.billable.project.BillingType;
import ee.ta25.billable.project.Project;
import ee.ta25.billable.project.ProjectRepository;
import ee.ta25.billable.project.ProjectService;
import java.time.Clock;
import java.time.LocalDateTime;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

/** R9: when a capped project's value reaches 80 % of its cap, warn the owner once. */
@Service
public class BudgetMonitor {

  static final int WARNING_THRESHOLD_PERCENT = 80;

  private final ProjectRepository projects;
  private final ProjectService projectService;
  private final HourlyBilling hourlyBilling;
  private final Notifier notifier;
  private final Clock clock;

  public BudgetMonitor(
      ProjectRepository projects,
      ProjectService projectService,
      HourlyBilling hourlyBilling,
      Notifier notifier,
      Clock clock) {
    this.projects = projects;
    this.projectService = projectService;
    this.hourlyBilling = hourlyBilling;
    this.notifier = notifier;
    this.clock = clock;
  }

  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void check(long projectId) {
    Project project = projects.findById(projectId).orElseThrow();

    if (project.getBillingType() != BillingType.CAPPED) {
      return;
    }
    if (project.getBudgetWarningSentAt() != null) {
      return;
    }
    if (project.getBudgetCapCents() == null || project.getBudgetCapCents() <= 0) {
      return;
    }

    Money cap = Money.fromCents(project.getBudgetCapCents());
    // The UNCAPPED value, from the same minutes the project page uses.
    Money value =
        hourlyBilling.calculate(project, projectService.billableMinutes(project), Money.zero());
    if (!reachedThreshold(value, cap)) {
      return;
    }

    // R9: claim the warning in the database first. Only one caller gets 1 back.
    if (projects.claimBudgetWarning(projectId, LocalDateTime.now(clock)) != 1) {
      return;
    }
    try {
      notifier.notify(
          "Budget warning: " + project.getName(),
          "Project \"%s\" has reached %d %% of its budget cap: %s of %s."
              .formatted(
                  project.getName(),
                  value.cents() * 100 / cap.cents(),
                  value.format(),
                  cap.format()));
    } catch (RuntimeException e) {
      projects.releaseBudgetWarning(projectId); // not sent, so not claimed: a later check retries
      throw e;
    }
  }

  /** True when {@code value} is at least 80 % of {@code cap} (R9). Exact, no rounding. */
  public static boolean reachedThreshold(Money value, Money cap) {
    return value.multiply(100).greaterThanOrEqual(cap.multiply(WARNING_THRESHOLD_PERCENT));
  }
}
```

**What happens — the collaborators, and which double each one gets:**

| Collaborator | What `check()` uses it for | In the test |
|---|---|---|
| `ProjectRepository` | load the project; claim the warning (and release it on failure) | **stub**: `findById` returns our test project, `claimBudgetWarning` returns `1` ("you won") or `0` ("someone else did"); `verify` for `releaseBudgetWarning` |
| `ProjectService` | billable minutes (lesson 10: rounded per entry) | **stub**: `billableMinutes` returns e.g. 80 |
| `HourlyBilling` | the uncapped value | **real object** — it is pure |
| `Notifier` | the warning | **mock**: verify it was (or was not) called |
| `Clock` | the timestamp | **stub**: `Clock.fixed(...)` |

The claim is an `UPDATE` in the database (lesson 09, Step 9), so the timestamp is not set on the
`Project` object any more. With a mocked repository, "the timestamp was stored" becomes "the claim was
made with exactly this timestamp".

Only the boundaries are doubles. The calculation in the middle (`HourlyBilling`, `Money`,
`reachedThreshold`) is real, so the test still checks the arithmetic.

`ProjectService` is a big class with its own repositories. Here it is only "the thing that knows the
minutes", so a stub is the simplest honest double: `billableMinutes` itself is tested elsewhere (it is a
query — lesson 14 tests it against PostgreSQL).

---

## Step 3 — More builder methods in `ProjectTestData`

**Why:** the new tests need a project **with an id**, an **archived** project and a project whose
warning was **already sent**. We add those to the builder from lesson 11, so tests stay one-liners.

Replace the file with this version (the new parts are the three fields and methods marked `NEW`, and the
end of `build()`):

```java
// src/test/java/ee/ta25/billable/project/ProjectTestData.java
package ee.ta25.billable.project;

import ee.ta25.billable.client.Client;
import java.time.LocalDateTime;
import org.springframework.test.util.ReflectionTestUtils;

/**
 * Builds {@link Project} objects for unit tests, without a database. Every value has a sensible
 * default, so a test only sets what it is about.
 */
public final class ProjectTestData {

  private Long id; // NEW
  private String name = "Website redesign";
  private BillingType billingType = BillingType.HOURLY;
  private Long hourlyRateCents = 6000L;
  private Long fixedPriceCents;
  private Long budgetCapCents;
  private int roundingMinutes = 1;
  private LocalDateTime archivedAt; // NEW
  private LocalDateTime budgetWarningSentAt; // NEW

  private ProjectTestData() {}

  public static ProjectTestData hourly(long hourlyRateCents) {
    var data = new ProjectTestData();
    data.billingType = BillingType.HOURLY;
    data.hourlyRateCents = hourlyRateCents;
    return data;
  }

  public static ProjectTestData capped(long hourlyRateCents, long budgetCapCents) {
    var data = new ProjectTestData();
    data.billingType = BillingType.CAPPED;
    data.hourlyRateCents = hourlyRateCents;
    data.budgetCapCents = budgetCapCents;
    return data;
  }

  public static ProjectTestData fixed(long fixedPriceCents) {
    var data = new ProjectTestData();
    data.billingType = BillingType.FIXED;
    data.hourlyRateCents = null;
    data.fixedPriceCents = fixedPriceCents;
    return data;
  }

  public ProjectTestData named(String name) {
    this.name = name;
    return this;
  }

  public ProjectTestData rounding(int roundingMinutes) {
    this.roundingMinutes = roundingMinutes;
    return this;
  }

  /** NEW: the id the database would have given the project. */
  public ProjectTestData withId(long id) {
    this.id = id;
    return this;
  }

  /** NEW */
  public ProjectTestData archivedAt(LocalDateTime archivedAt) {
    this.archivedAt = archivedAt;
    return this;
  }

  /** NEW */
  public ProjectTestData budgetWarningSentAt(LocalDateTime budgetWarningSentAt) {
    this.budgetWarningSentAt = budgetWarningSentAt;
    return this;
  }

  /** The only place in the tests that knows how a Project is constructed. */
  public Project build() {
    var client = new Client("Test client OÜ", "client@example.com", null);
    var project = new Project(client, name, billingType, roundingMinutes);
    project.setHourlyRateCents(hourlyRateCents);
    project.setFixedPriceCents(fixedPriceCents);
    project.setBudgetCapCents(budgetCapCents);
    if (archivedAt != null) {
      project.archive(archivedAt);
    }
    project.setBudgetWarningSentAt(budgetWarningSentAt);
    if (id != null) {
      ReflectionTestUtils.setField(project, "id", id);
    }
    return project;
  }
}
```

**What happens:**

1. In production the **database** assigns the id (`@GeneratedValue`), and `Project` has no `setId()` —
   correctly, application code must never change an id. A test without a database still needs one,
   because `TimeEntryService` publishes `new TimeEntryLoggedEvent(project.getId())`.
2. `ReflectionTestUtils.setField(object, "id", value)` (from `spring-test`) writes a private field by
   name using reflection. Use it **only** in test data builders, for fields the database would normally
   fill. If the field name changes, the builder fails loudly: `Could not find field 'id'`.
3. We do **not** add a `setId()` to the entity just for tests. Production code should not get worse to
   make tests easier.

**Check it works:** `./mvnw test-compile` → `BUILD SUCCESS`. Lesson 11's tests still pass.

---

## Step 4 — The first Mockito test: the warning is sent at 80 %

**Why:** this is the central behaviour of R9. Choose numbers you can check in your head: 60 €/h is 1 €
per minute; a cap of 100 € makes **every minute exactly 1 % of the cap**. 80 minutes = 80 %.

```java
// src/test/java/ee/ta25/billable/budget/BudgetMonitorTest.java
package ee.ta25.billable.budget;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import ee.ta25.billable.billing.HourlyBilling;
import ee.ta25.billable.notification.Notifier;
import ee.ta25.billable.project.Project;
import ee.ta25.billable.project.ProjectRepository;
import ee.ta25.billable.project.ProjectService;
import ee.ta25.billable.project.ProjectTestData;
import java.time.Clock;
import java.time.Instant;
import java.time.LocalDateTime;
import java.time.ZoneId;
import java.util.Optional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Captor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class BudgetMonitorTest {

  private static final long PROJECT_ID = 7L;

  // 11:30 UTC is 14:30 in Tallinn (summer time, UTC+3, until 25 October 2026).
  private static final Clock CLOCK =
      Clock.fixed(Instant.parse("2026-10-05T11:30:00Z"), ZoneId.of("Europe/Tallinn"));
  private static final LocalDateTime NOW = LocalDateTime.of(2026, 10, 5, 14, 30);

  @Mock private ProjectRepository projects;
  @Mock private ProjectService projectService;
  @Mock private Notifier notifier;

  @Captor private ArgumentCaptor<String> subject;

  private BudgetMonitor monitor;

  @BeforeEach
  void setUp() {
    monitor = new BudgetMonitor(projects, projectService, new HourlyBilling(), notifier, CLOCK);
  }

  @Test
  void warns_the_owner_when_the_project_reaches_80_percent() {
    cappedProjectWithMinutes(80);
    when(projects.claimBudgetWarning(PROJECT_ID, NOW)).thenReturn(1); // this call wins the claim

    monitor.check(PROJECT_ID);

    verify(notifier).notify(subject.capture(), anyString());
    assertThat(subject.getValue()).contains("Website redesign");
  }

  /** 60 €/h and a 100 € cap: every logged minute is exactly 1 % of the cap. */
  private Project cappedProjectWithMinutes(long minutes) {
    var project = ProjectTestData.capped(6000, 10_000).withId(PROJECT_ID).build();
    when(projects.findById(PROJECT_ID)).thenReturn(Optional.of(project));
    when(projectService.billableMinutes(project)).thenReturn(minutes);
    return project;
  }
}
```

**What happens:**

1. `@ExtendWith(MockitoExtension.class)` lets Mockito take part in the JUnit test life cycle. Before each
   test it creates a fresh mock for every field annotated with `@Mock`, and a fresh captor for every
   `@Captor`. No Spring is involved — this is still a plain unit test.
2. A **mock** created by Mockito implements the type (interface or class) and does nothing: methods
   return `null`, `0`, `false`, empty collections or `Optional.empty()`, until you tell them otherwise.
3. `setUp()` builds the monitor **by hand**, passing the mocks, a **real** `HourlyBilling` and the fixed
   clock. We do not use Mockito's `@InjectMocks`: the clock and the real `HourlyBilling` are not mocks,
   and an explicit constructor call fails to compile the day the constructor changes — a clear signal,
   instead of a silent `null`.
4. `Clock.fixed(instant, zone)` is a **stub** for time: it always returns the same instant.
   `LocalDateTime.now(clock)` inside `check()` therefore gives 2026-10-05 14:30 in Tallinn. The comment
   explains the time-zone arithmetic, because that is where readers stumble.
5. **Arrange.** `cappedProjectWithMinutes(80)` builds a real `Project` with the builder and **stubs** two
   calls: `when(projects.findById(7L)).thenReturn(Optional.of(project))` — "when somebody asks for project
   7, give them this one" — and `projectService.billableMinutes(project)` returns 80. The test itself
   stubs the claim: `claimBudgetWarning(7L, NOW)` returns `1`, as the database would for the first
   request. Without this stub the mock returns `0` (the default for `int`), the monitor thinks another
   request won, and the test fails with "Wanted but not invoked".
6. **Act.** `monitor.check(7L)` runs the real code. It loads "project 7" (our object), gets 80 minutes,
   calculates 80,00 € with the real `HourlyBilling`, compares with 100,00 €, claims the warning and
   calls `notify(...)`.
7. **Assert — interaction.** `verify(notifier).notify(subject.capture(), anyString())` checks that
   `notify` was called **exactly once** (the default of `verify`) and captures the first argument.
   `anyString()` is a matcher for the second argument: we do not care about the exact message text.
8. **Assert — the captured value.** `subject.getValue()` is the real string the code passed. We check it
   with normal AssertJ: it names the project.
9. **The timestamp.** The claim stub only matches `NOW` — exactly 2026-10-05 14:30, thanks to the
   fixed clock. If `check()` used `LocalDateTime.now()` without the clock, it would call
   `claimBudgetWarning(7L, <some other time>)`, and strict stubs (step 5) would fail the test with
   `PotentialStubbingProblem`. So the stub also checks *which* timestamp is stored. We do not add a
   separate `verify(projects).claimBudgetWarning(...)`: a stub that must match is already the check.
   (In production the claim is an `UPDATE` in the database; the `Project` object is not changed, so
   there is no state on it to assert.)

**Check it works:**

```bash
./mvnw test -Dtest=BudgetMonitorTest
```

→ `Tests run: 1, Failures: 0`.

**See it fail.** Comment out the `notifier.notify(...)` call in `BudgetMonitor` and run again:

```text
Wanted but not invoked:
notifier.notify(<Capturing argument: String>, <any string>);
-> at ee.ta25.billable.budget.BudgetMonitorTest.warns_the_owner_when_the_project_reaches_80_percent
Actually, there were zero interactions with this mock.
```

Undo the change.

**Commit:** `test(budget): warning is sent at 80 % with a mocked notifier and fixed clock`

---

## Step 5 — Proving that nothing happened

**Why:** R9 says "once", and only for capped projects at 80 % or more. Most of these are tests that
something did **not** happen (README, section 4). The last one checks what happens when sending fails.

Add these tests to `BudgetMonitorTest`, above the private helper. New static imports:
`org.assertj.core.api.Assertions.assertThatThrownBy`, `org.mockito.ArgumentMatchers.any`,
`org.mockito.ArgumentMatchers.anyLong`, `org.mockito.Mockito.doThrow`, `org.mockito.Mockito.never` and
`org.mockito.Mockito.verifyNoInteractions`.

```java
// src/test/java/ee/ta25/billable/budget/BudgetMonitorTest.java — add inside the class
  @Test
  void does_not_warn_at_79_percent() {
    cappedProjectWithMinutes(79);

    monitor.check(PROJECT_ID);

    verify(notifier, never()).notify(anyString(), anyString());
    verify(projects, never()).claimBudgetWarning(anyLong(), any());
  }

  @Test
  void does_not_warn_again_when_a_warning_was_already_sent() {
    var sentEarlier = LocalDateTime.of(2026, 10, 1, 9, 0);
    var project =
        ProjectTestData.capped(6000, 10_000)
            .withId(PROJECT_ID)
            .budgetWarningSentAt(sentEarlier)
            .build();
    when(projects.findById(PROJECT_ID)).thenReturn(Optional.of(project));

    monitor.check(PROJECT_ID);

    verifyNoInteractions(notifier);
    verify(projects, never()).claimBudgetWarning(anyLong(), any());
  }

  @Test
  void does_not_warn_for_an_hourly_project() {
    var project = ProjectTestData.hourly(6000).withId(PROJECT_ID).build();
    when(projects.findById(PROJECT_ID)).thenReturn(Optional.of(project));

    monitor.check(PROJECT_ID);

    verifyNoInteractions(notifier);
  }

  @Test
  void does_not_warn_when_another_request_claimed_the_warning_first() {
    cappedProjectWithMinutes(85);
    when(projects.claimBudgetWarning(PROJECT_ID, NOW)).thenReturn(0); // the row was already set

    monitor.check(PROJECT_ID);

    verifyNoInteractions(notifier);
  }

  @Test
  void releases_the_claim_when_sending_fails() {
    cappedProjectWithMinutes(85);
    when(projects.claimBudgetWarning(PROJECT_ID, NOW)).thenReturn(1);
    doThrow(new IllegalStateException("Mail server down"))
        .when(notifier)
        .notify(anyString(), anyString());

    assertThatThrownBy(() -> monitor.check(PROJECT_ID)).hasMessage("Mail server down");

    verify(projects).releaseBudgetWarning(PROJECT_ID);
  }
```

**What happens:**

1. `verify(notifier, never()).notify(anyString(), anyString())` — "notify was called zero times, with any
   arguments". `never()` is the same as `times(0)`.
2. `verifyNoInteractions(notifier)` is stronger: **no method at all** was called on the mock. Use it when
   the mock should not have been touched in any way.
3. 79 % and 80 % (step 4) are the **boundary pair**. Together they pin the comparison to `>=` at exactly
   80 %.
4. The "already sent" test and the hourly test **do not stub `billableMinutes`**. The monitor returns
   early, before it asks for minutes. If we stubbed it anyway, Mockito would fail the test with
   `UnnecessaryStubbingException`. That is Mockito's **strict stubs** mode (the default with
   `MockitoExtension`): it keeps tests honest — every stub must be needed. Here it even documents the
   behaviour: "for an hourly project, the minutes are never loaded".
5. In the 79 % and "already sent" tests, `verify(projects, never()).claimBudgetWarning(anyLong(), any())`
   checks that the timestamp is **not touched**. A bug that skips the email but still claims (and so
   overwrites the old timestamp) would be caught. `anyLong()` and `any()` match every id and every time.
6. **Another request was first.** `claimBudgetWarning` returns `0`: the database found the column
   already set. That is the race from lesson 09 — the loaded `project` still says `null`, but another
   request has claimed the warning in the meantime. `0` is also what an unstubbed `int` method returns;
   we stub it anyway, so the test *says* which scenario it is about.
7. **Sending fails.** `doThrow(...).when(notifier).notify(...)` makes the mock throw — the stub form
   for `void` methods, because `when(notifier.notify(...))` does not compile for a method without a
   return value. `assertThatThrownBy` checks that `check()` re-throws, and
   `verify(projects).releaseBudgetWarning(7L)` that the claim was given back, so a later entry tries
   again. This is a stub (input: "the mail server is down") and a mock check (output: "release was
   called") in one test.

**Check it works:** `./mvnw test -Dtest=BudgetMonitorTest` → `Tests run: 6`.

**See one fail — or not.** Comment out the `if (project.getBudgetWarningSentAt() != null)` block in
`check()` and run the tests. Surprise: `does_not_warn_again_when_a_warning_was_already_sent` **still
passes**. Read the code to see why: the monitor now goes on and calls `projectService.billableMinutes(...)`.
That method is not stubbed, so the mock returns the default value for `long`, which is `0`. Zero minutes
is below 80 %, so no warning — for the wrong reason. This test alone does not prove the "once" rule. That
is a real finding, and a typical trap with mocks: unstubbed methods quietly return "nothing". Step 6
closes the gap. Undo the change.

**Commit:** `test(budget): no-warning cases and release of the claim on failure`

---

## Step 6 — A hand-written fake: `InMemoryNotifier`

**Why:** the gap from step 5: "once" must hold even when the project is **above** 80 %, where nothing
else stops the second warning. The most realistic test is "check twice, get one warning". A fake that
collects messages makes that easy to write and to read (README, section 10).

```java
// src/test/java/ee/ta25/billable/notification/InMemoryNotifier.java
package ee.ta25.billable.notification;

import java.util.ArrayList;
import java.util.List;

/** Test fake: remembers every notification instead of sending it. */
public class InMemoryNotifier implements Notifier {

  /** One notification as it was received. */
  public record Sent(String subject, String message) {}

  private final List<Sent> sent = new ArrayList<>();

  @Override
  public void notify(String subject, String message) {
    sent.add(new Sent(subject, message));
  }

  public List<Sent> sent() {
    return List.copyOf(sent);
  }
}
```

A second test class, so its setup can use the fake instead of the mock:

```java
// src/test/java/ee/ta25/billable/budget/BudgetMonitorWithFakeNotifierTest.java
package ee.ta25.billable.budget;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.when;

import ee.ta25.billable.billing.HourlyBilling;
import ee.ta25.billable.notification.InMemoryNotifier;
import ee.ta25.billable.project.ProjectRepository;
import ee.ta25.billable.project.ProjectService;
import ee.ta25.billable.project.ProjectTestData;
import java.time.Clock;
import java.time.Instant;
import java.time.LocalDateTime;
import java.time.ZoneId;
import java.util.Optional;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class BudgetMonitorWithFakeNotifierTest {

  private static final Clock CLOCK =
      Clock.fixed(Instant.parse("2026-10-05T11:30:00Z"), ZoneId.of("Europe/Tallinn"));

  @Mock private ProjectRepository projects;
  @Mock private ProjectService projectService;

  private final InMemoryNotifier notifier = new InMemoryNotifier();

  @Test
  void a_second_check_above_80_percent_sends_no_second_warning() {
    var project = ProjectTestData.capped(6000, 10_000).withId(7L).build();
    when(projects.findById(7L)).thenReturn(Optional.of(project));
    when(projectService.billableMinutes(project)).thenReturn(85L);
    // Like the database: the first claim changes the row, every later one finds it already set.
    when(projects.claimBudgetWarning(eq(7L), any(LocalDateTime.class))).thenReturn(1, 0);
    var monitor = new BudgetMonitor(projects, projectService, new HourlyBilling(), notifier, CLOCK);

    monitor.check(7L);
    monitor.check(7L);

    assertThat(notifier.sent()).hasSize(1);
    assertThat(notifier.sent().getFirst().subject()).isEqualTo("Budget warning: Website redesign");
    assertThat(notifier.sent().getFirst().message()).contains("85 %");
  }
}
```

**What happens:**

1. The fake is in **test** sources, in the `notification` package next to the interface. It is a real
   `Notifier`: `notify()` stores a `Sent` record in a list.
2. `sent()` returns an **unmodifiable copy** (`List.copyOf`), so a test cannot accidentally change the
   fake's history.
3. The repository and `ProjectService` are still Mockito stubs — the fake replaces only the notifier. Mixing
   is normal: use whichever double fits each collaborator.
4. **The claim stub plays the database.** `thenReturn(1, 0)` answers `1` on the first call and `0` on
   every call after that — exactly what the conditional `UPDATE` does: the first claim changes the row,
   the second finds the timestamp already set. `eq(7L)` and `any(LocalDateTime.class)` are matchers; once
   one argument uses a matcher, all of them must (see Troubleshooting).
5. **Act twice.** `findById` returns the **same object** both times, and it still says
   `budgetWarningSentAt = null`: the monitor never changes the object, the claim is an `UPDATE`. So the
   second `check()` gets past the quick "already sent" check — as in the race from lesson 09 — and is
   stopped by the claim result `0`.
6. **Assert** with plain AssertJ: one message, its subject, a meaningful part of its text (85 % —
   `8500 × 100 / 10000`). `getFirst()` is the Java 21 `List` method.
7. Now see it fail: in `BudgetMonitor`, ignore the claim result — call
   `projects.claimBudgetWarning(...)` on its own line and send anyway. This test fails with
   `Expected size: 1 but was: 2` — the gap is closed. Undo the change. (Removing the quick "already
   sent" check, as in step 5, makes no test fail, and that is right: it only saves a query, the claim
   still guarantees "once".)

**Mock or fake — compare** this test with step 4 and 5:

| | Mock (`@Mock Notifier`) | Fake (`InMemoryNotifier`) |
|---|---|---|
| Where is the check? | `verify(...)` after the act | Plain assertions on `sent()` |
| What can you check? | What matchers and captors describe | Anything: count, order, full content |
| Failure message | `Wanted but not invoked` / `wanted 1 time but was 2` | `Expected size: 1 but was: 2` plus the list contents |
| Reuse | Configured again in each test | One class for every test |
| Best for | "Never called", one precise interaction | Scenarios with several calls; inspecting content |

**Check it works:** `./mvnw test -Dtest='BudgetMonitor*'` → 7 tests.

**Commit:** `test(budget): InMemoryNotifier fake; once-only warning above 80 %`

---

## Step 7 — `TimeEntryService`: events, rejections and a fixed clock

**Why:** `TimeEntryService` is the **publisher** of `TimeEntryLoggedEvent` (lesson 09) and it owns two
rules: no time on archived projects, no entries that end in the future (lesson 07). We test all of that
without a database.

The "future" rule and the `Clock` parameter come from the lesson 07 intermediate task. If you skipped
it, add `TimeEntryInFutureException`, the `Clock clock` constructor parameter (before
`ApplicationEventPublisher events`) and the `isAfter(LocalDateTime.now(clock))` check in `log()` from
that solution now — `ClockConfig` already provides the `Clock` bean since lesson 09.

```java
// src/test/java/ee/ta25/billable/timeentry/TimeEntryServiceTest.java
package ee.ta25.billable.timeentry;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.verifyNoInteractions;
import static org.mockito.Mockito.when;

import ee.ta25.billable.budget.TimeEntryLoggedEvent;
import ee.ta25.billable.project.ProjectArchivedException;
import ee.ta25.billable.project.ProjectRepository;
import ee.ta25.billable.project.ProjectTestData;
import java.time.Clock;
import java.time.Instant;
import java.time.LocalDateTime;
import java.time.ZoneId;
import java.util.Optional;
import java.util.Set;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Captor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.context.ApplicationEventPublisher;

@ExtendWith(MockitoExtension.class)
class TimeEntryServiceTest {

  // 15:00 UTC is 18:00 in Tallinn.
  private static final Clock CLOCK =
      Clock.fixed(Instant.parse("2026-10-05T15:00:00Z"), ZoneId.of("Europe/Tallinn"));
  private static final LocalDateTime NOW = LocalDateTime.of(2026, 10, 5, 18, 0);
  private static final long PROJECT_ID = 7L;

  @Mock private TimeEntryRepository timeEntries;
  @Mock private ProjectRepository projects;
  @Mock private TagRepository tags;
  @Mock private ApplicationEventPublisher events;

  @Captor private ArgumentCaptor<TimeEntryLoggedEvent> publishedEvent;

  private TimeEntryService service;

  @BeforeEach
  void setUp() {
    service = new TimeEntryService(timeEntries, projects, tags, CLOCK, events);
  }

  @Test
  void logging_an_entry_saves_it_and_publishes_time_entry_logged() {
    var project = ProjectTestData.hourly(6000).withId(PROJECT_ID).build();
    when(projects.findById(PROJECT_ID)).thenReturn(Optional.of(project));
    when(timeEntries.save(any(TimeEntry.class))).thenAnswer(invocation -> invocation.getArgument(0));

    var entry = service.log(command(NOW.minusHours(2), NOW.minusMinutes(30)));

    assertThat(entry.getProject()).isSameAs(project);
    assertThat(entry.getDescription()).isEqualTo("Wireframes for the start page");
    verify(events).publishEvent(publishedEvent.capture());
    assertThat(publishedEvent.getValue().projectId()).isEqualTo(PROJECT_ID);
  }

  @Test
  void an_archived_project_is_rejected_and_nothing_is_saved_or_published() {
    var project =
        ProjectTestData.hourly(6000).withId(PROJECT_ID).archivedAt(NOW.minusDays(5)).build();
    when(projects.findById(PROJECT_ID)).thenReturn(Optional.of(project));

    assertThatThrownBy(() -> service.log(command(NOW.minusHours(2), NOW.minusMinutes(30))))
        .isInstanceOf(ProjectArchivedException.class);

    verify(timeEntries, never()).save(any());
    verifyNoInteractions(events);
  }

  @Test
  void an_entry_that_ends_one_minute_in_the_future_is_rejected() {
    var project = ProjectTestData.hourly(6000).withId(PROJECT_ID).build();
    when(projects.findById(PROJECT_ID)).thenReturn(Optional.of(project));

    assertThatThrownBy(() -> service.log(command(NOW.minusHours(1), NOW.plusMinutes(1))))
        .isInstanceOf(TimeEntryInFutureException.class);

    verify(timeEntries, never()).save(any());
    verifyNoInteractions(events);
  }

  private static LogTimeEntryCommand command(LocalDateTime startedAt, LocalDateTime endedAt) {
    return new LogTimeEntryCommand(
        PROJECT_ID, startedAt, endedAt, "Wireframes for the start page", true, Set.of());
  }
}
```

**What happens:**

1. Four collaborators are mocks. That sounds like a lot, but they are all **boundaries**: three
   repositories (the database) and the event publisher (the framework). `TimeEntryService` has no pure
   logic of its own besides its rules — which is exactly what we test.
2. `when(timeEntries.save(any(TimeEntry.class))).thenAnswer(invocation -> invocation.getArgument(0))`: an
   unstubbed `save()` on a mock returns `null`, and the service would return `null`. `thenAnswer` runs a
   small function for each call; here it returns the argument unchanged — "saving returns the same
   entity", as `save()` does for a new entity in production.
3. `tags.findAllById(Set.of())` is **not** stubbed: an unstubbed method that returns a `List` gives an
   empty list, which is what "no tags selected" means.
4. `verify(events).publishEvent(publishedEvent.capture())` captures the event. We then check its
   content: the event is about project 7. Checking only "an event was published" would miss a bug that
   publishes the wrong id.
5. The rejection tests check **three** things: the exception type, that nothing was saved
   (`verify(..., never()).save(any())`), and that no event was published (`verifyNoInteractions(events)`).
   The last one matters most: an event for a rejected entry would make `BudgetMonitor` run for nothing —
   or worse, if a later listener sends an email.
6. The fixed clock makes "one minute in the future" **exact**: `NOW.plusMinutes(1)` is 18:01 and "now"
   is 18:00, on every machine, on every day. With `Clock.systemDefaultZone()` this test would depend on
   when you run it.

**Why `verifyNoInteractions(events)` and not `verify(events, never()).publishEvent(any())`?**
`ApplicationEventPublisher` has **two** `publishEvent` methods: one for `ApplicationEvent` and one for
`Object`. With `any()`, Java picks the more specific one (`ApplicationEvent`) — but our record is not an
`ApplicationEvent`, so the service calls the `Object` version. The `never()` check would then look at the
wrong method and pass even if the event **was** published. `verifyNoInteractions` avoids the trap. (If you
need `never()`, write `publishEvent(any(Object.class))`.)

**Check it works:**

```bash
./mvnw test -Dtest=TimeEntryServiceTest
```

→ `Tests run: 3`.

**Commit:** `test(time-entries): events, archived and future rules with mocks and a fixed clock`

---

## Step 8 — A web slice test: `@WebMvcTest(ClientController.class)`

**Why:** so far we tested services with plain `new`. A controller is different: its behaviour is
mostly **Spring's** — URL mapping, binding form fields into `ClientForm`, running Jakarta Validation,
choosing a view. We want to test that wiring **without** the database. `@WebMvcTest` starts only the
web layer, and `@MockitoBean` puts a mock `ClientService` into that small Spring context.

```java
// src/test/java/ee/ta25/billable/client/ClientControllerTest.java
package ee.ta25.billable.client;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.verifyNoInteractions;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;

import ee.ta25.billable.common.SecurityConfig;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest;
import org.springframework.context.annotation.Import;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.assertj.MockMvcTester;

@WebMvcTest(ClientController.class)
@Import(SecurityConfig.class)
class ClientControllerTest {

  @Autowired private MockMvcTester mvc;

  @MockitoBean private ClientService clients;

  @Test
  void shows_the_form_again_with_errors_when_the_input_is_invalid() {
    var result =
        mvc.post()
            .uri("/clients")
            .with(csrf())
            .formField("name", "")
            .formField("email", "not-an-email")
            .exchange();

    assertThat(result).hasStatusOk().hasViewName("clients/create");
    assertThat(result).model().extractingBindingResult("form").hasFieldErrors("name", "email");
    verifyNoInteractions(clients);
  }
}
```

**What happens:**

1. `@WebMvcTest(ClientController.class)` creates a **slice** of the application: the Spring MVC
   infrastructure, Thymeleaf, validation, `@ControllerAdvice` classes (like `FormBindingAdvice` from
   lesson 05) — and only this one controller. No JPA, no Flyway, no database, no other services. It
   starts in about a second instead of several.
2. Because `ClientService` is not in the slice, the controller's constructor would fail with *No qualifying
   bean of type 'ClientService'*. `@MockitoBean private ClientService clients` creates a Mockito mock and
   registers it **as a bean** in the test context, so Spring injects it into the controller. It is reset
   after every test.
3. `@Autowired MockMvcTester mvc` — `MockMvcTester` is the modern, AssertJ-based entry point to MockMvc.
   It sends requests **directly into Spring MVC**, without a real HTTP server or port. (Tests field
   injection with `@Autowired` is normal in test classes; the "constructor injection only" rule is for
   production code.)
4. `mvc.post().uri("/clients").formField(...)` builds a form POST like the browser sends
   (`application/x-www-form-urlencoded`). `.exchange()` performs the request and returns the result.
5. **Security in the slice.** `@WebMvcTest` also switches on Spring Security, but it does **not** load
   our `SecurityConfig` (a slice only picks up web classes such as controllers and advice, not
   `@Configuration` classes). Without `@Import(SecurityConfig.class)` the test would get Boot's default
   security, which wants a login: `401` instead of our page. With the import, the slice has the same
   rules as the app: everyone may enter, and every POST needs a CSRF token.
6. `.with(csrf())` adds that token, like the hidden `_csrf` field in the browser's form. It comes from
   `spring-boot-starter-security-test` (lesson 05). Without it the answer is `403 Forbidden`.
   `MockMvcTester` requests accept the same `with(...)` post-processors as classic `MockMvc`.
7. `hasStatusOk().hasViewName("clients/create")`: on validation errors the controller re-renders the
   form (status 200) instead of redirecting.
8. `model().extractingBindingResult("form").hasFieldErrors("name", "email")` looks inside the
   `BindingResult` of the model attribute `form` — the same errors the template shows under the fields.
9. `verifyNoInteractions(clients)` — the service was never called. Invalid input stops **at the edge**,
   in the web layer. This is the web-layer version of "prove that nothing happened".

**Check it works:**

```bash
./mvnw test -Dtest=ClientControllerTest
```

→ `Tests run: 1`. In the log you see the slice start with only a handful of beans. If it fails with a
Thymeleaf error, the template references something the controller did not put into the model — the test
has found a real problem in the error path.

**Commit:** `test(clients): web slice test for invalid client form`

---

## Step 9 — Run everything

```bash
./mvnw spotless:apply
./mvnw verify
```

**Check it works:** `BUILD SUCCESS`. Surefire reports the new classes:

```text
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0 -- in ee.ta25.billable.budget.BudgetMonitorTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0 -- in ee.ta25.billable.budget.BudgetMonitorWithFakeNotifierTest
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0 -- in ee.ta25.billable.timeentry.TimeEntryServiceTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0 -- in ee.ta25.billable.client.ClientControllerTest
```

Open the JaCoCo report from lesson 11 (`target/site/jacoco/index.html`): `budget`, `timeentry` and
`client` now have coverage too — without a database.

Push; CI runs `./mvnw verify` as before.

**Commit:** `test: budget, time entry and controller tests with doubles`

---

## Troubleshooting

| Error or symptom | Cause | Fix |
|---|---|---|
| `org.mockito.exceptions.misusing.UnnecessaryStubbingException: Unnecessary stubbings detected` | A `when(...)` that the test never uses (strict stubs) | Remove the stub. If the code really may or may not call it, the test scenario is unclear — split it. As a last resort `lenient().when(...)` |
| `PotentialStubbingProblem: Strict stubbing argument mismatch` | The code called a stubbed method with **different** arguments (e.g. `findById(8L)` instead of `7L`) | Check the ids; the test found a real mismatch |
| `Wanted but not invoked: notifier.notify(...)` / `Actually, there were zero interactions with this mock.` | The code returned early | Check your arrange: capped? cap set? enough minutes? `claimBudgetWarning` stubbed to return `1`? |
| `InvalidUseOfMatchersException: Invalid use of argument matchers! 2 matchers expected, 1 recorded` | Mixed a matcher and a plain value: `notify("x", anyString())` | All matchers: `notify(eq("x"), anyString())` |
| `NullPointerException` inside the service during a Mockito test | An unstubbed method returned `null` (e.g. `save()`), or a constructor parameter is `null` | Stub it (`thenAnswer(i -> i.getArgument(0))`); check the order of constructor arguments in `setUp()` |
| `verify(events, never()).publishEvent(any())` passes although an event is published | Overload trap: `any()` selects `publishEvent(ApplicationEvent)` | `verifyNoInteractions(events)` or `publishEvent(any(Object.class))` |
| `cannot find symbol: class MockBean` / `WebMvcTest` | Boot 3 imports from a tutorial | `@MockitoBean` from `org.springframework.test.context.bean.override.mockito`; `WebMvcTest` from `org.springframework.boot.webmvc.test.autoconfigure` |
| `package org.springframework.boot.webmvc.test.autoconfigure does not exist` | `spring-boot-starter-webmvc-test` missing | Add it (step 1) |
| `@WebMvcTest`: status `401` (or a redirect to `/login`) instead of the page | Boot's default security is active because `SecurityConfig` is not in the slice | `@Import(SecurityConfig.class)` on the test class |
| `@WebMvcTest`: a POST returns `403` | No CSRF token | `.with(csrf())` from `SecurityMockMvcRequestPostProcessors` |
| `NoSuchBeanDefinitionException: No qualifying bean of type 'ee.ta25.billable.client.ClientService'` in `@WebMvcTest` | The controller's dependency is not in the slice | `@MockitoBean` for every constructor dependency of the controller |
| `@WebMvcTest` fails: `No qualifying bean of type ... Repository` or tries to connect to the database | A class in the slice (e.g. a `@ControllerAdvice` or a `WebMvcConfigurer`) depends on a repository, or the main class has `@EnableJpaRepositories` | Mock that dependency with `@MockitoBean`; keep JPA configuration out of the main application class |
| `Could not find field 'id' of type [null] on target object` | `ReflectionTestUtils.setField` with a wrong field name | Use the entity's real field name |
| `PotentialStubbingProblem` on `claimBudgetWarning`, with a time 2 or 3 hours away from `NOW` | `Clock.fixed` with `ZoneOffset.UTC`, or wrong zone | Use `ZoneId.of("Europe/Tallinn")` and remember UTC+3 in summer, UTC+2 in winter |
| `Mockito is currently self-attaching to enable the inline-mock-maker` warning | Java 21+ dynamic agent warning | Harmless; optional fix in step 1 |
| `Error: Could not find or load main class @{argLine}` | `@{argLine}` used but no plugin sets `argLine` | Keep JaCoCo's `prepare-agent`, or remove `@{argLine}` |

---

## Recap

- New test files: `budget/BudgetMonitorTest.java`, `budget/BudgetMonitorWithFakeNotifierTest.java`,
  `timeentry/TimeEntryServiceTest.java`, `client/ClientControllerTest.java`,
  `notification/InMemoryNotifier.java`; `project/ProjectTestData.java` extended.
- `@ExtendWith(MockitoExtension.class)` + `@Mock` / `@Captor` in plain unit tests; the class under test is
  built by hand in `@BeforeEach`.
- `when(...).thenReturn(...)` stubs input; `verify(...)`, `verify(..., never())`, `verifyNoInteractions`
  check output; `ArgumentCaptor` grabs the real argument for normal assertions.
- `Clock.fixed(...)` makes "now" a known value; the service uses `LocalDateTime.now(clock)`.
- Strict stubs fail on unused stubs — stub only what the scenario needs.
- The database claim is a stub too: `thenReturn(1)` ("you won"), `thenReturn(0)` ("someone else
  did"), `thenReturn(1, 0)` (first call wins); `doThrow(...).when(notifier)` simulates a failed send.
- A hand-written fake (`InMemoryNotifier`) closed a gap the mock-based "already sent" test left open.
- `@WebMvcTest` + `@MockitoBean` + `MockMvcTester` test the web layer without a database (Boot 4
  packages); `@Import(SecurityConfig.class)` brings our security rules into the slice, and every POST
  gets `.with(csrf())`.

---

## Independent work — solutions

> **The tasks are not here.** The task descriptions (Basic / Intermediate / Advanced, with acceptance
> criteria) are in this lesson's [README.md → Independent work (~2 h)](README.md#independent-work-2-h).
> Read them there first and build the features in your project. This section only contains the
> solutions — try each task yourself, then open a solution to compare or when you are stuck.

<details>
<summary><strong>Task 1 (Basic)</strong> — tests for every <code>TimeEntryService</code> rule</summary>

Add to `TimeEntryServiceTest`:

```java
// src/test/java/ee/ta25/billable/timeentry/TimeEntryServiceTest.java — add inside the class
  @Test
  void an_unknown_project_is_rejected_and_nothing_is_published() {
    when(projects.findById(PROJECT_ID)).thenReturn(Optional.empty());

    assertThatThrownBy(() -> service.log(command(NOW.minusHours(2), NOW.minusMinutes(30))))
        .isInstanceOf(NotFoundException.class)
        .hasMessageContaining("7");

    verifyNoInteractions(timeEntries, events);
  }

  @Test
  void the_selected_tags_are_attached_to_the_entry() {
    var project = ProjectTestData.hourly(6000).withId(PROJECT_ID).build();
    var design = new Tag("design");
    when(projects.findById(PROJECT_ID)).thenReturn(Optional.of(project));
    when(tags.findAllById(Set.of(3L))).thenReturn(List.of(design));
    when(timeEntries.save(any(TimeEntry.class))).thenAnswer(invocation -> invocation.getArgument(0));

    var entry =
        service.log(
            new LogTimeEntryCommand(
                PROJECT_ID, NOW.minusHours(2), NOW.minusMinutes(30), "Logo", true, Set.of(3L)));

    assertThat(entry.getTags()).containsExactly(design);
  }
```

(Imports: `ee.ta25.billable.common.NotFoundException`, `java.util.List`. `new Tag("design")` is the public `Tag(String name)` constructor from
lesson 06 — a `Tag` is plain data, do not mock it.)

**What happens:**

1. `Optional.empty()` is the stubbed answer for "no such project". Lesson 07's service throws
   `NotFoundException("Project 7 not found")`, which the web layer turns into a 404.
   `verifyNoInteractions(timeEntries, events)` checks two mocks at once.
2. The tag test stubs `findAllById` with the **exact** id set. If the service passed a different set,
   strict stubs would report `PotentialStubbingProblem` — a precise failure.
3. R4 (end after start, at most 12 hours) is validated on the **form** (`TimeEntryForm`, lesson 06), not
   in the service. If your service checks it too, add two more rejection tests in the same style.

Together with the three tests from class, every rule the service owns is covered, and every rejection
proves that nothing was saved and nothing was published.

**Commit:** `test(time-entries): cover all TimeEntryService rules`

</details>

<details>
<summary><strong>Task 2 (Basic)</strong> — the future-entry rule at its boundaries</summary>

The fixed clock says **18:00** in Tallinn. A parameterized test for the accepted rows, and the rejected
row from class:

```java
// src/test/java/ee/ta25/billable/timeentry/TimeEntryServiceTest.java — add inside the class
// imports: org.junit.jupiter.params.ParameterizedTest, org.junit.jupiter.params.provider.ValueSource
  @ParameterizedTest(name = "an entry ending {0} minute(s) before now is accepted")
  @ValueSource(longs = {1, 0})
  void an_entry_that_ends_by_now_is_accepted(long minutesBeforeNow) {
    var project = ProjectTestData.hourly(6000).withId(PROJECT_ID).build();
    when(projects.findById(PROJECT_ID)).thenReturn(Optional.of(project));
    when(timeEntries.save(any(TimeEntry.class))).thenAnswer(invocation -> invocation.getArgument(0));

    service.log(command(NOW.minusHours(1), NOW.minusMinutes(minutesBeforeNow)));

    verify(timeEntries).save(any(TimeEntry.class));
    verify(events).publishEvent(any(TimeEntryLoggedEvent.class));
  }
```

**What happens:** the service rejects when `endedAt.isAfter(LocalDateTime.now(clock))`. With the fixed
clock, "now" is always 2026-10-05T18:00. Ending at 17:59 (1 minute before) and at 18:00 exactly (0 minutes
before — equal is not "after") are accepted; 18:01 (`an_entry_that_ends_one_minute_in_the_future_is_rejected`
from class) is rejected. These are the three boundary rows from the README.

`verify(events).publishEvent(any(TimeEntryLoggedEvent.class))` uses a **typed** matcher. The argument type
`TimeEntryLoggedEvent` makes Java choose the `publishEvent(Object)` overload, which is the one the service
calls.

Try it: replace `CLOCK` in `setUp()` with `Clock.systemDefaultZone()`. The accepted rows now fail with
`TimeEntryInFutureException` (you are running this before 5 October 2026, 18:00), and they would start
to pass after that date — a test that changes its result with the calendar. Put the fixed clock back.

**Commit:** `test(time-entries): future-entry rule at its boundaries with a fixed clock`

</details>

<details>
<summary><strong>Task 3 (Intermediate)</strong> — <code>@WebMvcTest</code> for creating a client: valid, invalid, duplicate</summary>

The invalid case is the test from step 8. Add the other two to `ClientControllerTest`:

```java
// src/test/java/ee/ta25/billable/client/ClientControllerTest.java — add inside the class
// imports: static org.mockito.ArgumentMatchers.any, static org.mockito.Mockito.verify,
//          static org.mockito.Mockito.when, org.springframework.test.util.ReflectionTestUtils
  @Test
  void a_valid_client_is_created_and_the_browser_is_redirected_to_it() {
    var saved = new Client("Saar OÜ", "info@saar.ee", null);
    ReflectionTestUtils.setField(saved, "id", 42L);
    when(clients.create(any(ClientCommand.class))).thenReturn(saved);

    var result =
        mvc.post()
            .uri("/clients")
            .with(csrf())
            .formField("name", "  Saar OÜ ")
            .formField("email", "info@saar.ee")
            .exchange();

    assertThat(result).hasStatus3xxRedirection().hasRedirectedUrl("/clients/42");
    assertThat(result).flash().containsEntry("status", "Client Saar OÜ created.");
    verify(clients).create(new ClientCommand("Saar OÜ", "info@saar.ee", null));
  }

  @Test
  void a_duplicate_email_shows_the_form_again_with_an_error_on_email() {
    when(clients.create(any(ClientCommand.class)))
        .thenThrow(new DuplicateEmailException("info@saar.ee"));

    var result =
        mvc.post()
            .uri("/clients")
            .with(csrf())
            .formField("name", "Another Saar")
            .formField("email", "info@saar.ee")
            .exchange();

    assertThat(result).hasStatusOk().hasViewName("clients/create");
    assertThat(result).model().extractingBindingResult("form").hasFieldErrors("email");
  }
```

**What happens:**

1. **Valid.** The mock service returns a `Client` with id 42 (set with `ReflectionTestUtils`, like the
   project builder). The controller redirects to `/clients/42` and adds a flash attribute —
   `flash().containsEntry(...)` reads it.
2. `verify(clients).create(new ClientCommand("Saar OÜ", "info@saar.ee", null))` compares with
   `equals()`. `ClientCommand` is a **record**, so two commands with the same values are equal — no
   captor needed. It also proves the whitespace around the name was removed before the service was
   called (the compact constructor of `ClientCommand`, lesson 07).
3. **Duplicate.** `thenThrow(...)` stubs a failure: the service says "this e-mail is taken". The
   controller catches `DuplicateEmailException`, adds a field error on `email` and shows the form again.
   The **real** duplicate check (`existsByEmailIgnoreCase`) is not tested here — this is a web-layer test.
   It belongs in a `ClientService` unit test or a database test (lesson 14).
4. Compare with the Laravel track, where the same scenarios run as feature tests against a test database
   without any mock. Both are valid: `@WebMvcTest` is faster and more focused; a full feature test also
   covers the unique query.

**Commit:** `test(clients): web slice tests for creating a client`

</details>

<details>
<summary><strong>Task 4 (Advanced)</strong> — refactor an over-mocked test into a fake-based one</summary>

**The over-mocked version** — for reading, do not add it to your project:

```java
// An over-mocked test: the entity, the pure calculator and the notifier are all mocks
@Test
void warns_at_80_percent() {
  Project project = mock(Project.class);
  when(project.getId()).thenReturn(7L);
  when(project.getName()).thenReturn("Website redesign");
  when(project.getBillingType()).thenReturn(BillingType.CAPPED);
  when(project.getBudgetWarningSentAt()).thenReturn(null);
  when(project.getBudgetCapCents()).thenReturn(10_000L);
  when(projects.findById(7L)).thenReturn(Optional.of(project));
  when(projectService.billableMinutes(project)).thenReturn(80L);
  HourlyBilling hourly = mock(HourlyBilling.class);
  when(hourly.calculate(project, 80L, Money.zero())).thenReturn(Money.fromCents(8000));
  when(projects.claimBudgetWarning(7L, NOW)).thenReturn(1);

  new BudgetMonitor(projects, projectService, hourly, notifier, CLOCK).check(7L);

  verify(notifier)
      .notify(
          "Budget warning: Website redesign",
          "Project \"Website redesign\" has reached 80 % of its budget cap: 80,00 € of 100,00 €.");
  verifyNoMoreInteractions(notifier, hourly);
}
```

What is wrong with it:

- The **entity** is a mock with five stubbed getters. A real `Project` from `ProjectTestData` is one line.
- `HourlyBilling` is **pure** and is mocked anyway, with the answer (8000) typed in by hand. The test no
  longer checks that 80 minutes at 60 €/h is 80 € — it checks that the code calls a method.
- The notifier must receive the **complete** message text. Changing "reached" to "used" breaks the test,
  although R9 still works.
- `verifyNoMoreInteractions` makes any additional harmless call a failure.

**The fake-based version** is `a_second_check_above_80_percent_sends_no_second_warning` from step 6
(plus the claim stubbed with the exact `NOW` from step 4): a real project, a real
`HourlyBilling`, the `InMemoryNotifier`, and assertions on what the owner would notice.

Example text for `docs/testing.md`:

```markdown
<!-- docs/testing.md — add this section -->
## Mock or fake?

My first `BudgetMonitor` test mocked the `Project` entity (five getters), the pure `HourlyBilling`
calculator and the `Notifier`, and verified the complete message text plus
`verifyNoMoreInteractions`. It passed, but it described the implementation line by line: rewording
the message, or calling `HourlyBilling.forMinutes()` directly, broke it although the rule R9 still
held. And because `HourlyBilling` was mocked, the test no longer checked the amount at all.

The new test builds a real project with `ProjectTestData`, uses the real `HourlyBilling`, and a
hand-written `InMemoryNotifier` fake that stores every message in a list. It calls `check()` twice
and asserts that exactly one warning was sent and that its subject names the project. It checks what
the owner would notice, it is shorter, and it caught a gap the mock-based "already sent" test missed
(see step 5 of lesson 12).

I kept Mockito stubs for `ProjectRepository` and `ProjectService`: they stand for the database,
and a stub is the simplest way to say "this project exists and has 85 minutes". I also kept a mock
with `verifyNoInteractions` for the hourly-project test, because there the behaviour is exactly one
interaction that must not happen.
```

**Commit:** `test(budget): replace an over-mocked test with a fake; document the choice`

</details>
