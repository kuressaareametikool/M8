# 14 — Integration tests and data integrity · Spring Boot

[← Concepts](README.md) · Starting point: end of lesson 13 · Estimated time in class: 2 h

## What we build today

- Testcontainers (set up in lesson 01) now used for repository and integration tests: every database
  test runs against a real, throw-away PostgreSQL 17
- `TestDatabase` — a small JDBC helper to create test data without depending on entity constructors
- `@DataJpaTest` repository tests: the "already billed" aggregate query, the invoiceable-entries
  query, and an **N+1 check** with Hibernate `Statistics`
- `@SpringBootTest` + `MockMvcTester` end-to-end test of the invoice flow
- A **concurrency test** with `ExecutorService` + `CountDownLatch` that generates 10 invoices in
  parallel — it fails with lesson 13's "max + 1", and passes after the fix
- `invoice_sequences` + `@Lock(LockModeType.PESSIMISTIC_WRITE)`: gap-free, race-free numbering
- `@Version` on `Invoice`, and a clear message when two people change the same invoice at once

**Final result:** `./mvnw test` starts one PostgreSQL container, runs unit, slice and end-to-end tests
against it, and proves that ten parallel invoices get the numbers `2026-0001` to `2026-0010`.

---

## Step 1 — What you already have

**Why:** Flyway's PostgreSQL scripts, `ddl-auto=validate`, `ON CONFLICT` and row locks only work on
PostgreSQL. You have had a real, throw-away PostgreSQL for tests since lesson 01:
`TestcontainersConfiguration` with a `@ServiceConnection` `PostgreSQLContainer`, used by
`contextLoads` locally and in CI. Until now only that one smoke test used it. Today we use the same
container for repository and integration tests. Docker Desktop must be running.

Check that `pom.xml` has these test dependencies (scope `test`, no `<version>`):

| Dependency | Since | Gives us |
|---|---|---|
| `spring-boot-testcontainers`, `testcontainers-junit-jupiter`, `testcontainers-postgresql` | lesson 01 | the container and `@ServiceConnection` |
| `spring-boot-starter-webmvc-test` | lesson 01 (lesson 11 showed it) | `@AutoConfigureMockMvc`, `MockMvcTester` |
| `spring-boot-starter-security-test` | lesson 05 | `csrf()` for form posts in tests |
| `spring-boot-starter-data-jpa-test` | lesson 01 | `@DataJpaTest` |

In Boot 4 every feature starter has its own test starter; Initializr added the JPA one together with
Spring Data JPA. If `spring-boot-starter-data-jpa-test` is missing, add it like the others.

**Check it works:** `./mvnw test` passes, and the log shows `Container postgres:17-alpine started`.

---

## Step 2 — A fixed clock and a test-data helper

`TestcontainersConfiguration` stays as it is since lesson 01 (one `@ServiceConnection`
`PostgreSQLContainer("postgres:17-alpine")` bean in a `public` `@TestConfiguration` class). It is
`public` because today's tests in other packages, such as `ee.ta25.billable.invoice`, import it. Next to
it we add two small helpers.

```java
// src/test/java/ee/ta25/billable/FixedClockConfiguration.java
package ee.ta25.billable;

import ee.ta25.billable.common.ClockConfig;
import java.time.Clock;
import java.time.Instant;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Primary;

/** "Now" is 30 September 2026, 12:00 in Tallinn, for every test that imports this. */
@TestConfiguration(proxyBeanMethods = false)
public class FixedClockConfiguration {

  @Bean
  @Primary
  Clock fixedClock() {
    return Clock.fixed(Instant.parse("2026-09-30T09:00:00Z"), ClockConfig.ZONE);
  }
}
```

Test data is inserted with plain SQL. That keeps the tests independent of entity constructors, and it
is exactly what the database will contain.

```java
// src/test/java/ee/ta25/billable/TestDatabase.java
package ee.ta25.billable;

import java.time.LocalDateTime;
import java.util.concurrent.atomic.AtomicInteger;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.test.jdbc.JdbcTestUtils;

/** Creates and removes test rows with plain SQL. */
public class TestDatabase {

  private final JdbcTemplate jdbc;
  private final AtomicInteger counter = new AtomicInteger();

  public TestDatabase(JdbcTemplate jdbc) {
    this.jdbc = jdbc;
  }

  /** Removes all business data; TRUNCATE … CASCADE follows every foreign key to clients. */
  public void clear() {
    jdbc.execute("truncate table clients restart identity cascade");
  }

  public long client(String name) {
    return jdbc.queryForObject(
        """
        insert into clients (name, email, created_at, updated_at)
        values (?, ?, now(), now()) returning id
        """,
        Long.class,
        name,
        "client" + counter.incrementAndGet() + "@example.com");
  }

  public long hourlyProject(long clientId, String name, long rateCents, int roundingMinutes) {
    return jdbc.queryForObject(
        """
        insert into projects (client_id, name, billing_type, hourly_rate_cents, rounding_minutes,
                              created_at, updated_at)
        values (?, ?, 'hourly', ?, ?, now(), now()) returning id
        """,
        Long.class,
        clientId,
        name,
        rateCents,
        roundingMinutes);
  }

  public long entry(long projectId, String start, String end, String description) {
    return jdbc.queryForObject(
        """
        insert into time_entries (project_id, started_at, ended_at, description, billable,
                                  created_at, updated_at)
        values (?, ?, ?, ?, true, now(), now()) returning id
        """,
        Long.class,
        projectId,
        LocalDateTime.parse(start),
        LocalDateTime.parse(end),
        description);
  }

  public long invoice(long clientId, String number, String status) {
    return jdbc.queryForObject(
        """
        insert into invoices (client_id, number, status, issued_on, due_on, vat_rate_bp,
                              subtotal_cents, vat_cents, total_cents, created_at, updated_at)
        values (?, ?, ?, date '2026-09-30', date '2026-10-14', 2400, 0, 0, 0, now(), now())
        returning id
        """,
        Long.class,
        clientId,
        number,
        status);
  }

  public void line(long invoiceId, long projectId, long amountCents) {
    jdbc.update(
        """
        insert into invoice_lines (invoice_id, project_id, description, minutes, amount_cents)
        values (?, ?, 'Line', 60, ?)
        """,
        invoiceId,
        projectId,
        amountCents);
  }

  public Long invoiceIdOf(long entryId) {
    return jdbc.queryForObject(
        "select invoice_id from time_entries where id = ?", Long.class, entryId);
  }

  public int count(String table) {
    return JdbcTestUtils.countRowsInTable(jdbc, table);
  }
}
```

**What happens:**

1. A reminder from lesson 01: `@TestConfiguration` is configuration that only tests import. The
   `PostgreSQLContainer` bean is started when the test context starts, and stopped when the JVM ends.
2. `@ServiceConnection` reads the container's random port, user and password and sets up the
   `DataSource` with them. Flyway then runs all your migrations on the empty container, and
   `ddl-auto=validate` checks the entities.
3. Spring **caches** test contexts: every test class with the same configuration reuses the same
   context — and therefore the same container. That is why we will give all `@SpringBootTest` classes
   the same annotations (Step 4).
4. Docker Compose support (lesson 01) is **skipped in tests** by default, so your development database
   is never touched by tests. `DevDataSeeder` has `@Profile("dev")`, so it does not run either.
5. `FixedClockConfiguration` adds a second `Clock` bean marked `@Primary`, so everything that injects
   `Clock` gets the fixed one. Its bean name (`fixedClock`) differs from the one in `ClockConfig`, so
   Spring does not complain about overriding a bean.
6. In `TestDatabase`, `returning id` makes PostgreSQL send back the new id, and `queryForObject` reads
   it. Every `?` is a **bind parameter** — never string concatenation (README, concept 9).
   `count(String table)` uses Spring's `JdbcTestUtils.countRowsInTable()`; a table name cannot be a bind
   parameter, so it only takes names from our own test code.
7. `truncate table clients restart identity cascade` empties `clients` and **every table that refers to
   it**, directly or indirectly (projects, time entries, their tags links, invoices, invoice lines), and
   resets the id counters.

> **Built-in alternatives.** Spring's test support has `@Sql("/invoice-fixtures.sql")` to run a SQL
> file before a test, and `JdbcTestUtils` for counting and deleting rows. Since Spring 6.1 there is
> also `JdbcClient`, a fluent wrapper around `JdbcTemplate`
> (`jdbc.sql("…").params(…).query(Long.class).single()`). We use a small helper class with
> `JdbcTemplate` because each test needs different ids (the `returning id` values), which a fixed SQL
> file cannot give back — and every Spring tutorial you will read uses `JdbcTemplate`.

---

## Step 3 — Repository tests with `@DataJpaTest`

**Why:** the aggregate query and the entries query from lesson 13 were mocked in the Mockito test. Here
we prove they return the right rows from a real database — and that loading entries with their projects
takes **one** SQL statement.

```java
// src/test/java/ee/ta25/billable/invoice/InvoiceQueriesTest.java
package ee.ta25.billable.invoice;

import static org.assertj.core.api.Assertions.assertThat;

import ee.ta25.billable.TestDatabase;
import ee.ta25.billable.TestcontainersConfiguration;
import ee.ta25.billable.client.Client;
import ee.ta25.billable.client.ClientRepository;
import ee.ta25.billable.project.BillingType;
import ee.ta25.billable.timeentry.TimeEntry;
import ee.ta25.billable.timeentry.TimeEntryRepository;
import jakarta.persistence.EntityManager;
import jakarta.persistence.EntityManagerFactory;
import java.time.LocalDateTime;
import java.util.List;
import org.hibernate.SessionFactory;
import org.hibernate.stat.Statistics;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest;
import org.springframework.context.annotation.Import;
import org.springframework.jdbc.core.JdbcTemplate;

@DataJpaTest(properties = "spring.jpa.properties.hibernate.generate_statistics=true")
@Import(TestcontainersConfiguration.class)
class InvoiceQueriesTest {

  @Autowired private JdbcTemplate jdbc;
  @Autowired private EntityManager em;
  @Autowired private EntityManagerFactory emf;
  @Autowired private InvoiceLineRepository invoiceLines;
  @Autowired private TimeEntryRepository timeEntries;
  @Autowired private ClientRepository clients;

  private TestDatabase db;

  @BeforeEach
  void setUp() {
    db = new TestDatabase(jdbc);
  }

  @Test
  void sums_already_billed_amounts_per_requested_project_only() {
    long client = db.client("Saare Web OÜ");
    long website = db.hourlyProject(client, "Website", 6000, 1);
    long shop = db.hourlyProject(client, "Shop", 6000, 1);
    long other = db.hourlyProject(client, "Other", 6000, 1);
    long first = db.invoice(client, "2026-0001", "sent");
    long second = db.invoice(client, "2026-0002", "draft");
    db.line(first, website, 1_000);
    db.line(second, website, 2_500);
    db.line(second, shop, 700);
    db.line(second, other, 9_999);

    List<ProjectAmount> sums = invoiceLines.sumAmountByProject(List.of(website, shop));

    assertThat(sums)
        .containsExactlyInAnyOrder(
            new ProjectAmount(website, 3_500L), new ProjectAmount(shop, 700L));
  }

  @Test
  void finds_only_billable_uninvoiced_entries_of_the_client_up_to_the_boundary() {
    long client = db.client("Saare Web OÜ");
    long otherClient = db.client("Someone else");
    long project = db.hourlyProject(client, "Website", 6000, 15);
    long foreign = db.hourlyProject(otherClient, "Foreign", 6000, 15);
    long wanted = db.entry(project, "2026-09-30T09:00", "2026-09-30T23:59", "Wanted");
    db.entry(project, "2026-09-30T23:30", "2026-10-01T00:30", "Ends tomorrow");
    db.entry(foreign, "2026-09-29T09:00", "2026-09-29T10:00", "Other client");
    long invoiced = db.entry(project, "2026-09-28T09:00", "2026-09-28T10:00", "Invoiced");
    jdbc.update(
        "update time_entries set invoice_id = ? where id = ?",
        db.invoice(client, "2026-0001", "sent"),
        invoiced);
    Client saare = clients.findById(client).orElseThrow();

    List<TimeEntry> found =
        timeEntries.findInvoiceable(
            saare, LocalDateTime.parse("2026-10-01T00:00"), BillingType.INTERNAL);

    assertThat(found).extracting(TimeEntry::getId).containsExactly(wanted);
  }

  @Test
  void loads_entries_with_their_projects_in_one_statement() {
    long client = db.client("Saare Web OÜ");
    for (int i = 1; i <= 5; i++) {
      long project = db.hourlyProject(client, "Project " + i, 6000, 1);
      String day = "2026-09-2" + i;
      db.entry(project, day + "T09:00", day + "T10:00", "E" + i);
    }
    Client saare = clients.findById(client).orElseThrow();
    em.clear();
    Statistics statistics = emf.unwrap(SessionFactory.class).getStatistics();
    statistics.clear();

    List<TimeEntry> found =
        timeEntries.findInvoiceable(
            saare, LocalDateTime.parse("2026-10-01T00:00"), BillingType.INTERNAL);
    found.forEach(entry -> entry.getProject().getName());

    assertThat(found).hasSize(5);
    assertThat(statistics.getPrepareStatementCount()).isEqualTo(1);
  }

  @Test
  void the_time_sheet_query_count_does_not_grow_with_the_data() {
    long client = db.client("Saare Web OÜ");
    createEntries(client, 3);
    long few = timeSheetStatements();

    createEntries(client, 12);
    long many = timeSheetStatements();

    assertThat(many).as("statements for 3 vs 15 entries").isEqualTo(few);
  }

  private void createEntries(long client, int count) {
    for (int i = 0; i < count; i++) {
      long project = db.hourlyProject(client, "P" + System.nanoTime(), 6000, 1);
      db.entry(project, "2026-09-29T09:00", "2026-09-29T10:00", "Entry");
    }
  }

  /** Loads the time sheet and touches everything the page shows. */
  private long timeSheetStatements() {
    em.clear();
    Statistics statistics = emf.unwrap(SessionFactory.class).getStatistics();
    statistics.clear();
    timeEntries
        .findAllForTimeSheet()
        .forEach(
            entry -> {
              entry.getProject().getClient().getName();
              entry.getTags().size();
            });
    return statistics.getPrepareStatementCount();
  }
}
```

**What happens:**

1. `@DataJpaTest` starts only the persistence part of the application: entities, repositories, the
   `DataSource`, `JdbcTemplate`, Flyway. No controllers, no services. Each test runs in a transaction that
   is **rolled back** at the end, so tests do not see each other's data.
2. `@Import(TestcontainersConfiguration.class)` brings in the container. `@DataJpaTest` replaces a
   *normal* `DataSource` with an embedded in-memory database (H2), but since Boot 3.4 it keeps one that
   is already a **test** database — and a `@ServiceConnection` container counts as one. So the slice
   runs on PostgreSQL with no extra annotation. Older tutorials add
   `@AutoConfigureTestDatabase(replace = Replace.NONE)` for this; it still works (in Boot 4 from
   `org.springframework.boot.jdbc.test.autoconfigure`), but it is no longer needed.
3. Flyway runs in the slice too, as long as `spring-boot-starter-flyway` is in the `pom.xml` (Boot 4
   needs the Boot Flyway module, not only `flyway-core`). Missing tables in a slice test almost always
   mean that module is missing.
4. `JdbcTemplate` inserts run in the **same** transaction as the JPA queries (the JPA transaction manager
   shares its connection), so the repository sees the rows.
5. The aggregate test puts a line for a third project in the table and checks it is **not** included.
   `ProjectAmount` is a record, so `containsExactlyInAnyOrder` compares by value.
6. The boundary test checks every condition of the query in one go: another client's entry, an
   invoiced entry and an entry ending after midnight are all excluded.
7. `generate_statistics=true` makes Hibernate count what it does. `em.clear()` empties the persistence
   context first — otherwise the entities could come from memory and the count would prove nothing.
   `getPrepareStatementCount()` is the number of SQL statements sent.
8. `loads_entries_with_their_projects_in_one_statement` proves `join fetch` works: 5 entries, 5
   different projects, **1** statement. Without the fetch it would be 6 (the classic N+1).
9. The time-sheet test uses your lesson 06 query `findAllForTimeSheet()` (ids with a limit, then the
   entities by id with `@EntityGraph` — two statements). It touches the project, client and tags of every
   entry, like the page does. The number must be the same for 3 and 15 entries. If your page shows other
   relations, touch those too.

**Check it works:** `./mvnw test -Dtest=InvoiceQueriesTest`. You see the container start and Flyway
applying your migrations in the log, and
`Tests run: 4, Failures: 0`. Remove `join fetch` from `findInvoiceable` for a moment: the N+1 test fails
with `expected: 1 but was: 6`.

**Commit:** `test(invoices): repository integration tests on PostgreSQL`

---

## Step 4 — The invoice flow end-to-end

**Why:** `@SpringBootTest` starts the **whole** application — controllers, services, templates,
transactions — against the container. `MockMvcTester` sends requests to it without a real HTTP port.

A small base class keeps the annotations identical for every end-to-end test, so they all share one
cached context and one container:

```java
// src/test/java/ee/ta25/billable/IntegrationTest.java
package ee.ta25.billable;

import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.webmvc.test.autoconfigure.AutoConfigureMockMvc;
import org.springframework.context.annotation.Import;
import org.springframework.jdbc.core.JdbcTemplate;

/** Base class for tests that need the whole application and a real database. */
@SpringBootTest
@AutoConfigureMockMvc
@Import({TestcontainersConfiguration.class, FixedClockConfiguration.class})
public abstract class IntegrationTest {

  @Autowired protected JdbcTemplate jdbc;

  protected TestDatabase db;

  @BeforeEach
  void cleanDatabase() {
    db = new TestDatabase(jdbc);
    db.clear();
  }
}
```

```java
// src/test/java/ee/ta25/billable/invoice/InvoiceFlowTest.java
package ee.ta25.billable.invoice;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;

import ee.ta25.billable.IntegrationTest;
import java.util.Map;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.web.servlet.assertj.MockMvcTester;
import org.springframework.test.web.servlet.assertj.MvcTestResult;

class InvoiceFlowTest extends IntegrationTest {

  @Autowired private MockMvcTester mvc;

  private long clientId;
  private long projectId;

  @BeforeEach
  void createClientAndProject() {
    clientId = db.client("Saare Web OÜ");
    projectId = db.hourlyProject(clientId, "Website redesign", 6000, 15);
  }

  @Test
  void generating_an_invoice_over_http() {
    long first = db.entry(projectId, "2026-09-28T09:00", "2026-09-28T10:10", "Design");
    long second = db.entry(projectId, "2026-09-29T13:00", "2026-09-29T14:00", "Build");

    assertThat(generate())
        .hasStatus3xxRedirection()
        .redirectedUrl()
        .matchesAntPattern("/invoices/*");

    Map<String, Object> invoice = jdbc.queryForMap("select * from invoices");
    assertThat(invoice)
        .containsEntry("number", "2026-0001")
        .containsEntry("status", "draft")
        .containsEntry("subtotal_cents", 13_500L)
        .containsEntry("vat_cents", 3_240L)
        .containsEntry("total_cents", 16_740L);
    long invoiceId = (Long) invoice.get("id");
    assertThat(db.invoiceIdOf(first)).isEqualTo(invoiceId);
    assertThat(db.invoiceIdOf(second)).isEqualTo(invoiceId);

    assertThat(mvc.get().uri("/invoices/{id}", invoiceId))
        .hasStatusOk()
        .bodyText()
        .contains("2026-0001", "Website redesign — 2 h 15 min");
  }

  @Test
  void the_second_invoice_gets_the_next_number() {
    db.entry(projectId, "2026-09-28T09:00", "2026-09-28T10:00", "First");
    generate();
    db.entry(projectId, "2026-09-29T09:00", "2026-09-29T10:00", "Second");
    generate();

    assertThat(jdbc.queryForList("select number from invoices order by id", String.class))
        .containsExactly("2026-0001", "2026-0002");
  }

  @Test
  void overlapping_entries_are_rejected_with_both_entries_named() {
    db.entry(projectId, "2026-09-29T13:00", "2026-09-29T17:00", "API integration");
    db.entry(projectId, "2026-09-29T13:30", "2026-09-29T14:00", "Client call");

    assertThat(generate())
        .hasStatusOk()
        .hasViewName("invoices/new")
        .bodyText()
        .contains("API integration", "Client call");
    assertThat(db.count("invoices")).isZero();
  }

  @Test
  void deleting_a_draft_releases_its_entries() {
    long entry = db.entry(projectId, "2026-09-28T09:00", "2026-09-28T10:00", "Work");
    generate();
    long invoiceId = jdbc.queryForObject("select id from invoices", Long.class);

    assertThat(mvc.post().uri("/invoices/{id}/delete", invoiceId).with(csrf()))
        .hasRedirectedUrl("/invoices");

    assertThat(db.count("invoices")).isZero();
    assertThat(db.count("invoice_lines")).isZero();
    assertThat(db.invoiceIdOf(entry)).isNull();
  }

  @Test
  void a_sent_invoice_cannot_be_deleted() {
    long entry = db.entry(projectId, "2026-09-28T09:00", "2026-09-28T10:00", "Work");
    generate();
    long invoiceId = jdbc.queryForObject("select id from invoices", Long.class);
    assertThat(mvc.post().uri("/invoices/{id}/send", invoiceId).with(csrf()))
        .hasStatus3xxRedirection();

    assertThat(mvc.post().uri("/invoices/{id}/delete", invoiceId).with(csrf()))
        .hasRedirectedUrl("/invoices/" + invoiceId)
        .flash()
        .containsKey("error");

    assertThat(jdbc.queryForObject("select status from invoices", String.class)).isEqualTo("sent");
    assertThat(db.invoiceIdOf(entry)).isEqualTo(invoiceId);
  }

  private MvcTestResult generate() {
    return mvc.post()
        .uri("/invoices")
        .with(csrf())
        .param("clientId", String.valueOf(clientId))
        .param("upTo", "2026-09-30")
        .exchange();
  }
}
```

**What happens:**

1. `@SpringBootTest` builds the full application context. `@AutoConfigureMockMvc` (Boot 4 package
   `org.springframework.boot.webmvc.test.autoconfigure`) adds a `MockMvcTester` bean: requests go through
   the `DispatcherServlet`, validation, controllers and Thymeleaf, but without a network socket.
2. Unlike `@DataJpaTest`, a `@SpringBootTest` does **not** roll back: the service commits real
   transactions. That is what we want here (we test transactions!), and why the base class truncates the
   tables **before** each test.
3. `mvc.post().uri(...).with(csrf()).param(...)` builds a form post like the browser's. `exchange()`
   performs it and returns an `MvcTestResult` that AssertJ can check: `hasStatus3xxRedirection()`,
   `redirectedUrl().matchesAntPattern("/invoices/*")`, `hasViewName(...)`, `flash()`, `bodyText()`.
4. We read the database with `JdbcTemplate`, not through the entities: the test checks what is really
   stored. `queryForMap` gives `bigint` columns as `Long`, hence `13_500L`.
5. Every post has `.with(csrf())`. Spring Security (lesson 05) checks the CSRF token on every POST, also
   in tests; without a token the answer is `403`. `csrf()` from `spring-boot-starter-security-test`
   adds a valid token to the request, as the browser's hidden `_csrf` field does (README, concept 9).
6. The overlap test expects status 200 and the form view: the controller re-renders the form with the
   message, and the layout's flash box shows `error`.
7. The fixed `Clock` makes "today" 30 September 2026, so the numbers start with `2026-`.

**Check it works:** `./mvnw test -Dtest=InvoiceFlowTest` → `Tests run: 5, Failures: 0`. The context
start (a few seconds) happens once; the tests themselves are fast.

**Commit:** `test(invoices): end-to-end invoice flow with MockMvcTester`

### `contextLoads` and CI

`BillableApplicationTests.contextLoads()` (lesson 01) has its own annotations
(`@SpringBootTest @Import(TestcontainersConfiguration.class)`). That is a **different** configuration
from `IntegrationTest`, so Spring builds a second context for it — with a second container. Let it
extend the base class instead (same annotations, so it shares the cached context and the container):

```java
// src/test/java/ee/ta25/billable/BillableApplicationTests.java
package ee.ta25.billable;

import org.junit.jupiter.api.Test;

class BillableApplicationTests extends IntegrationTest {

  @Test
  void contextLoads() {}
}
```

The CI workflow from lesson 02 needs no change: it has no database service, and GitHub's
`ubuntu-latest` runners have Docker, so Testcontainers starts PostgreSQL there exactly as on your
laptop. `./mvnw -B verify` runs every test — the unit tests from lesson 11, the Mockito and
`@WebMvcTest` tests from lessons 12–13 (they need no database at all) and today's integration tests.

**Check it works:** stop your development database (`docker compose stop postgres`) and run
`./mvnw test` — everything passes. Push, and the CI run is green.

---

## Step 5 — Make the race visible: a concurrency test

**Why:** the defence question — *"What happens if two people generate an invoice at the same second?"*
— deserves a test, not a guess. Ten threads wait at a **latch** (a gate that opens for all of them at
once), then each generates an invoice for its own client.

```java
// src/test/java/ee/ta25/billable/invoice/InvoiceNumberingConcurrencyTest.java
package ee.ta25.billable.invoice;

import static org.assertj.core.api.Assertions.assertThat;

import ee.ta25.billable.IntegrationTest;
import java.time.LocalDate;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.stream.IntStream;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;

class InvoiceNumberingConcurrencyTest extends IntegrationTest {

  private static final int INVOICES = 10;

  @Autowired private InvoiceService invoiceService;

  @Test
  void invoices_generated_in_parallel_get_unique_sequential_numbers() throws Exception {
    List<Long> clientIds = new ArrayList<>();
    for (int i = 1; i <= INVOICES; i++) {
      long clientId = db.client("Client " + i);
      long projectId = db.hourlyProject(clientId, "Project " + i, 6000, 1);
      db.entry(projectId, "2026-09-29T09:00", "2026-09-29T10:00", "Work");
      clientIds.add(clientId);
    }

    CountDownLatch start = new CountDownLatch(1);
    List<Future<String>> futures = new ArrayList<>();
    try (ExecutorService pool = Executors.newFixedThreadPool(INVOICES)) {
      for (long clientId : clientIds) {
        futures.add(
            pool.submit(
                () -> {
                  start.await();
                  return invoiceService
                      .generate(clientId, LocalDate.parse("2026-09-30"))
                      .getNumber();
                }));
      }
      start.countDown();
    }

    List<String> numbers = new ArrayList<>();
    List<Throwable> failures = new ArrayList<>();
    for (Future<String> future : futures) {
      try {
        numbers.add(future.get());
      } catch (ExecutionException e) {
        failures.add(e.getCause());
      }
    }

    assertThat(failures).as("generations that failed").isEmpty();
    assertThat(numbers)
        .containsExactlyInAnyOrderElementsOf(
            IntStream.rangeClosed(1, INVOICES).mapToObj("2026-%04d"::formatted).toList());
  }
}
```

**What happens:**

1. The loop creates ten clients, each with one entry, through `TestDatabase` (committed immediately).
2. `Executors.newFixedThreadPool(10)` gives ten threads. Each task first calls `start.await()` and
   blocks. `start.countDown()` opens the gate: all ten call `invoiceService.generate(...)` at nearly the
   same moment, each in its **own** transaction on its own connection.
3. `try (ExecutorService pool = ...)` — since Java 19 an `ExecutorService` is `AutoCloseable`; `close()`
   waits until all tasks have finished.
4. `future.get()` returns the number, or throws `ExecutionException` wrapping what went wrong in the
   thread. We collect both, so the failure message lists **every** problem.
5. The assertion demands no failures and exactly `2026-0001` … `2026-0010`: unique **and** gap-free.

**Run it now, with lesson 13's "max + 1":**

```bash
./mvnw test -Dtest=InvoiceNumberingConcurrencyTest
```

It fails, typically like this (the exact number of failures changes from run to run):

```
[generations that failed]
Expecting empty but was: [org.springframework.dao.DataIntegrityViolationException: could not execute
statement [ERROR: duplicate key value violates unique constraint "invoices_number_key"
  Detail: Key (number)=(2026-0001) already exists.] ...
```

Several threads read the same `MAX(number)` and tried to insert the same number. The unique constraint
stopped the duplicates — but those users got an error. Save this output as "before" evidence. If it
passes by luck, run it again; with ten threads the race almost always shows.

Do not commit a failing test to your main branch; commit it together with the fix in Step 7.

---

## Step 6 — `invoice_sequences`

Use the **next free** Flyway version. If lesson 13 ended at `V8`, this is `V9` (if you did the lesson 13
discount task, which added `V9`, every number in this lesson is one higher).

```sql
-- src/main/resources/db/migration/V9__create_invoice_sequences.sql
create table invoice_sequences (
    year        integer primary key,
    last_number integer not null
);

-- Continue from the invoices that already exist, so the next number is max + 1.
insert into invoice_sequences (year, last_number)
select cast(substr(number, 1, 4) as integer), max(cast(substr(number, 6, 4) as integer))
from invoices
group by cast(substr(number, 1, 4) as integer);
```

```java
// src/main/java/ee/ta25/billable/invoice/InvoiceSequence.java
package ee.ta25.billable.invoice;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;

/** The last invoice number issued in one year. One row per year, locked while a number is taken. */
@Entity
@Table(name = "invoice_sequences")
public class InvoiceSequence {

  @Id private Integer year;

  @Column(name = "last_number", nullable = false)
  private int lastNumber;

  protected InvoiceSequence() {}

  public Integer getYear() {
    return year;
  }

  /** Moves the counter on by one and returns the new value. */
  int increment() {
    lastNumber++;
    return lastNumber;
  }
}
```

```java
// src/main/java/ee/ta25/billable/invoice/InvoiceSequenceRepository.java
package ee.ta25.billable.invoice;

import jakarta.persistence.LockModeType;
import java.util.Optional;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Lock;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

public interface InvoiceSequenceRepository extends JpaRepository<InvoiceSequence, Integer> {

  /** Creates the year's row if it is missing. Safe when two transactions do it at the same time. */
  @Modifying
  @Query(
      value =
          """
          insert into invoice_sequences (year, last_number) values (:year, 0)
          on conflict (year) do nothing
          """,
      nativeQuery = true)
  void createIfMissing(@Param("year") int year);

  /** Loads the year's row and locks it until the transaction ends (SELECT … FOR UPDATE). */
  @Lock(LockModeType.PESSIMISTIC_WRITE)
  Optional<InvoiceSequence> findByYear(int year);
}
```

**What happens:**

1. `year` is the primary key: one row per year, with the unique index that `on conflict (year)` needs.
2. The `insert … select` **backfills** the counter from existing invoices. Without it, the next invoice
   after the migration would be `2026-0001` again and hit the unique constraint.
3. `createIfMissing` is a **native** query because JPQL has no `ON CONFLICT`. `@Modifying` marks it as a
   write. Two transactions creating the row for a new year at the same time: the second waits for the
   first and then inserts nothing. A "find, and save a new one if empty" in Java would **not** be safe:
   both could find nothing and both insert (README, concept 6).
4. `@Lock(LockModeType.PESSIMISTIC_WRITE)` on a derived query makes Hibernate add a row lock to the
   `SELECT` — in the SQL log you see `for update` or `for no key update` (PostgreSQL's lighter variant,
   depending on the Hibernate version). Both make a second writer **wait**.
5. `increment()` is package-private: only the number generator may move the counter. Dirty checking
   writes the new value at commit.

---

## Step 7 — The new `InvoiceNumberGenerator`, and locked entries

Replace the class:

```java
// src/main/java/ee/ta25/billable/invoice/InvoiceNumberGenerator.java
package ee.ta25.billable.invoice;

import java.util.regex.Matcher;
import java.util.regex.Pattern;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

/**
 * Gives out invoice numbers in the form YYYY-NNNN, sequential per year, with no gaps and no
 * duplicates (R11) — also when invoices are generated at the same moment.
 *
 * <p>The year's row in invoice_sequences stays locked until the caller's transaction ends, so a
 * second caller waits and then continues from the committed value.
 */
@Component
public class InvoiceNumberGenerator {

  private static final Pattern NUMBER = Pattern.compile("^\\d{4}-(\\d{4})$");

  private final InvoiceSequenceRepository sequences;

  public InvoiceNumberGenerator(InvoiceSequenceRepository sequences) {
    this.sequences = sequences;
  }

  @Transactional(propagation = Propagation.MANDATORY)
  public String next(int year) {
    sequences.createIfMissing(year);
    InvoiceSequence sequence =
        sequences
            .findByYear(year)
            .orElseThrow(() -> new IllegalStateException("No invoice sequence for " + year));
    return format(year, sequence.increment());
  }

  static String format(int year, int sequence) {
    if (sequence < 1 || sequence > 9999) {
      throw new IllegalArgumentException(
          "Invoice sequence must be between 1 and 9999, got " + sequence);
    }
    return "%04d-%04d".formatted(year, sequence);
  }

  static int sequenceOf(String number) {
    Matcher matcher = NUMBER.matcher(number);
    if (!matcher.matches()) {
      throw new IllegalArgumentException("\"" + number + "\" is not an invoice number");
    }
    return Integer.parseInt(matcher.group(1));
  }
}
```

Remove `findMaxNumberLike` from `InvoiceRepository` — nothing uses it any more.

Then lock the entries being invoiced. Add one annotation to the query method in `TimeEntryRepository`:

```java
// src/main/java/ee/ta25/billable/timeentry/TimeEntryRepository.java (on findInvoiceable)
// imports: jakarta.persistence.LockModeType, org.springframework.data.jpa.repository.Lock
  @Lock(LockModeType.PESSIMISTIC_WRITE)
  @Query(
      """
      select e from TimeEntry e
      ...
```

**What happens:**

1. `Propagation.MANDATORY` means "there must already be a transaction; otherwise throw
   `IllegalTransactionStateException`". Without a transaction the lock would be released right after the
   `SELECT` and protect nothing. `InvoiceService.generate()` provides the transaction.
2. `createIfMissing` → `findByYear` (locked) → `increment()`. A second transaction for the same year
   blocks inside `findByYear` until the first commits. In READ COMMITTED it then reads the **new**
   `last_number`.
3. If saving the invoice fails, the whole transaction rolls back, including the counter — no gap.
4. The lesson 13 unit tests for `format()` and `sequenceOf()` still pass. The Mockito
   `InvoiceGeneratorTest` mocks `InvoiceNumberGenerator`, so its new constructor does not matter there.
5. The lock on `findInvoiceable` stops the **double-billing** race: a second request for the same client
   waits until the first commits; PostgreSQL then re-checks `invoice_id is null`, finds nothing, and the
   user sees `NothingToInvoiceException`. Because of `join fetch`, Hibernate may lock the fetched project
   rows too. That only makes someone logging time on the same project wait a few milliseconds.

**Check it works — the "after" evidence:**

```bash
./mvnw test -Dtest=InvoiceNumberingConcurrencyTest
```

`Tests run: 1, Failures: 0`, every time. Run the whole suite as well: `./mvnw test`.

Also add the new table to the clean-up in `TestDatabase.clear()`, so every test starts at `0001`:

```java
// src/test/java/ee/ta25/billable/TestDatabase.java (replace clear)
  public void clear() {
    jdbc.execute("truncate table clients, invoice_sequences restart identity cascade");
  }
```

**Commit:** `fix(invoices): gap-free numbering with a locked sequence row` (include the concurrency test)

---

## Step 8 — Optimistic locking with `@Version`

**Why:** pessimistic locks are right for the short numbering step. For "someone else changed this
invoice while I was looking at it", an **optimistic** check is better: no waiting, but a conflicting
write is detected instead of silently overwriting. Example: two people press *Mark as paid* on the same
invoice at the same moment. Both load it as `sent`, both change it to `paid`. Without a version, both
succeed and the second write hides the first (a lost update). With a version, the second one fails.

Next free Flyway version:

```sql
-- src/main/resources/db/migration/V10__add_version_to_invoices.sql
alter table invoices add column version integer not null default 0;
```

In `Invoice`, add the field (with the import `jakarta.persistence.Version`):

```java
// src/main/java/ee/ta25/billable/invoice/Invoice.java (add next to the other fields)
  @Version private int version;
```

In `InvoiceController`, catch the conflict in `send`, `markPaid` and `delete`. The `markPaid` version:

```java
// src/main/java/ee/ta25/billable/invoice/InvoiceController.java (replace markPaid)
// import org.springframework.dao.OptimisticLockingFailureException;
  @PostMapping("/{id}/mark-paid")
  public String markPaid(@PathVariable Long id, RedirectAttributes redirect) {
    try {
      Invoice invoice = invoices.markPaid(id);
      redirect.addFlashAttribute(
          "status", "Invoice %s was marked as paid.".formatted(invoice.getNumber()));
    } catch (InvoicingException e) {
      redirect.addFlashAttribute("error", e.getMessage());
    } catch (OptimisticLockingFailureException e) {
      redirect.addFlashAttribute(
          "error", "Someone else changed this invoice at the same moment. Check it and try again.");
    }
    return "redirect:/invoices/" + id;
  }
```

A test that makes the collision happen on purpose, with two transactions:

```java
// src/test/java/ee/ta25/billable/invoice/InvoiceOptimisticLockingTest.java
package ee.ta25.billable.invoice;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import ee.ta25.billable.IntegrationTest;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.dao.OptimisticLockingFailureException;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.TransactionDefinition;
import org.springframework.transaction.support.TransactionTemplate;

class InvoiceOptimisticLockingTest extends IntegrationTest {

  @Autowired private InvoiceRepository invoices;
  @Autowired private PlatformTransactionManager transactionManager;

  @Test
  void a_second_concurrent_change_is_detected_instead_of_lost() {
    long invoiceId = db.invoice(db.client("Saare Web OÜ"), "2026-0001", "sent");
    TransactionTemplate me = new TransactionTemplate(transactionManager);
    TransactionTemplate colleague = new TransactionTemplate(transactionManager);
    colleague.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);

    assertThatThrownBy(
            () ->
                me.executeWithoutResult(
                    status -> {
                      Invoice mine = invoices.findById(invoiceId).orElseThrow(); // version 0
                      colleague.executeWithoutResult(
                          other ->
                              invoices
                                  .findById(invoiceId)
                                  .orElseThrow()
                                  .changeStatus(InvoiceStatus.PAID)); // commits version 1
                      mine.changeStatus(InvoiceStatus.PAID); // based on the stale version 0
                    }))
        .isInstanceOf(OptimisticLockingFailureException.class);

    assertThat(jdbc.queryForObject("select version from invoices", Integer.class)).isEqualTo(1);
  }
}
```

(If you did lesson 13's Task 1, `changeStatus` is now `transitionTo` — use that.)

**What happens:**

1. `@Version` makes Hibernate add the version to every update:
   `update invoices set status = ?, version = 1 … where id = ? and version = 0`. If another transaction
   already moved the version on, the `WHERE` matches **0 rows**, and Hibernate throws. Spring translates
   it to `ObjectOptimisticLockingFailureException`, a subclass of `OptimisticLockingFailureException`.
2. In the test, `me` loads the invoice (version 0). Inside it, `colleague` runs in a **new** transaction
   (`REQUIRES_NEW`, its own connection), changes the same invoice and commits (version 1). Then `me`
   changes its stale copy; at commit its update finds no row with version 0 → exception.
3. The plain `SELECT` does not lock anything, so the colleague is never blocked — that is the optimistic
   part.
4. In the controller the exception arrives from the `@Transactional` proxy at commit, **after**
   `markPaid()` itself returned — so it is caught in the controller, where we turn it into a message.
5. The version is also why "mark paid twice" from two browser tabs can no longer silently do two
   updates.

**Check it works:** `./mvnw test` — all green. `./mvnw spotless:apply` then `./mvnw spotless:check`.

**Commit:** `feat(invoices): optimistic locking with @Version`

> **Retry?** Spring Framework 7 has `@Retryable` (`org.springframework.resilience.annotation`, switched
> on with `@EnableResilientMethods`) to re-run a method after a failure. For a user action like "mark
> paid" a retry would be wrong — the user should see what changed. Retrying makes sense for
> transactions without side effects, such as a serialization failure in a batch job.

---

## Troubleshooting

| Error / symptom | Cause | Fix |
|---|---|---|
| `Could not find a valid Docker environment` | Docker Desktop is not running | Start Docker Desktop; `docker ps` must work in a terminal |
| A POST in a test returns status `403` | No CSRF token: `.with(csrf())` is missing | Add `.with(csrf())` (static import from `SecurityMockMvcRequestPostProcessors`) to every POST |
| `'dependencies.dependency.version' for org.testcontainers:postgresql:jar is missing` | Boot 3 artifact names | `testcontainers-postgresql`, `testcontainers-junit-jupiter` (lesson 01, Step 2) |
| `Unable to determine Dialect without JDBC metadata` | Old `org.testcontainers.containers.PostgreSQLContainer` import; `@ServiceConnection` ignored it | `import org.testcontainers.postgresql.PostgreSQLContainer;` |
| `Failed to replace DataSource with an embedded database for tests` | `@Import(TestcontainersConfiguration.class)` missing: there is no test database to keep, and no H2 to replace it with | Add the import (Step 3) |
| `relation "invoices" does not exist` in a slice test | Flyway did not run: `spring-boot-starter-flyway` missing (only `flyway-core`) | Add the Boot Flyway starter |
| `No transaction is in progress` / `IllegalTransactionStateException: No existing transaction found for transaction marked with propagation 'mandatory'` | `next()` called outside a transaction | Call it only from `InvoiceGenerator`, inside `InvoiceService.generate()` |
| Numbers start at `2026-0003` in a test | Data left by another `@SpringBootTest` | `TestDatabase.clear()` in `@BeforeEach` must also truncate `invoice_sequences` |
| Concurrency test hangs | A thread waits on the latch forever because `countDown()` was never reached (exception while submitting) | Keep `start.countDown()` inside the `try`, after the loop |
| `HikariPool-1 - Connection is not available, request timed out` | More threads than pool connections, and each holds a lock while waiting | Keep threads ≤ 10 (Hikari's default pool size) or raise `spring.datasource.hikari.maximum-pool-size` in the test |
| `ObjectOptimisticLockingFailureException` in normal use, not only in conflicts | The `version` column was added without `default 0` and old rows have `NULL` | The migration must set `not null default 0` |
| Tests pass alone but not together | A `@SpringBootTest` class with **different** annotations started a second context without cleaning | Extend `IntegrationTest` everywhere |

---

## Recap

- **Testcontainers** with `@ServiceConnection` (since lesson 01): now every database test runs on
  PostgreSQL 17, never on your dev database.
- **`TestDatabase`** creates rows with bound SQL; `truncate … cascade` gives every end-to-end test a clean start.
- **`@DataJpaTest` + Testcontainers**: the aggregate query, the invoiceable-entries query, and N+1 checks with Hibernate `Statistics`.
- **`@SpringBootTest` + `MockMvcTester`**: the full invoice flow — numbers, totals, links, overlap message, draft deletion, R13.
- **Concurrency test** with `ExecutorService` + `CountDownLatch`: red with max + 1, green after the fix.
- **`invoice_sequences`**: `on conflict do nothing` + `@Lock(PESSIMISTIC_WRITE)` + `Propagation.MANDATORY`; entries locked too.
- **`@Version`** on `Invoice`; conflicts become a message instead of a lost update.

---

## Independent work — solutions

<details>
<summary>Task 1 — More end-to-end tests for the main flows (Basic)</summary>

The form field names (`projectId`, `startedAt`, …) must match your `TimeEntryForm`; the date format
must match its `@DateTimeFormat` (here the `datetime-local` format).

```java
// src/test/java/ee/ta25/billable/MainFlowsTest.java
package ee.ta25.billable;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.web.servlet.assertj.MockMvcTester;

class MainFlowsTest extends IntegrationTest {

  @Autowired private MockMvcTester mvc;

  @Test
  void a_logged_entry_appears_on_the_time_sheet() {
    long project = db.hourlyProject(db.client("Saare Web OÜ"), "Website", 6000, 1);

    assertThat(
            mvc.post()
                .uri("/time-entries")
                .with(csrf())
                .param("projectId", String.valueOf(project))
                .param("startedAt", "2026-09-30T09:00")
                .param("endedAt", "2026-09-30T10:30")
                .param("description", "Homepage layout")
                .param("billable", "true"))
        .hasRedirectedUrl("/time-entries");

    assertThat(mvc.get().uri("/time-entries")).hasStatusOk().bodyText().contains("Homepage layout");
  }

  @Test
  void only_a_sent_invoice_can_be_marked_paid() {
    long invoice = db.invoice(db.client("Saare Web OÜ"), "2026-0001", "draft");

    assertThat(mvc.post().uri("/invoices/{id}/mark-paid", invoice).with(csrf()))
        .flash()
        .containsKey("error");
    assertThat(mvc.post().uri("/invoices/{id}/send", invoice).with(csrf()))
        .flash()
        .containsKey("status");
    assertThat(mvc.post().uri("/invoices/{id}/mark-paid", invoice).with(csrf()))
        .flash()
        .containsKey("status");

    assertThat(jdbc.queryForObject("select status from invoices", String.class)).isEqualTo("paid");
  }

  @Test
  void an_invoiced_entry_cannot_be_deleted() {
    long client = db.client("Saare Web OÜ");
    long project = db.hourlyProject(client, "Website", 6000, 1);
    long entry = db.entry(project, "2026-09-29T09:00", "2026-09-29T10:00", "Work");
    long invoice = db.invoice(client, "2026-0001", "sent");
    jdbc.update("update time_entries set invoice_id = ? where id = ?", invoice, entry);

    assertThat(mvc.post().uri("/time-entries/{id}/delete", entry).with(csrf()))
        .flash()
        .containsKey("error");

    assertThat(db.count("time_entries")).isEqualTo(1);
  }
}
```

</details>

<details>
<summary>Task 2 — Indexes justified by EXPLAIN (Basic)</summary>

**1. Data.** In `psql` (`docker compose exec postgres psql -U billable -d billable`), 10,000 entries in
one statement with `generate_series` (development database only):

```sql
insert into time_entries (project_id, started_at, ended_at, description, billable, created_at, updated_at)
select (select min(id) from projects) + (g % (select count(*) from projects)),
       timestamp '2025-01-01 08:00' + g * interval '47 minutes',
       timestamp '2025-01-01 08:45' + g * interval '47 minutes',
       'Seeded entry ' || g, true, now(), now()
from generate_series(1, 10000) as g;
analyze;
```

(This assumes your project ids have no gaps. If they do, pick ids with a sub-select from `projects`.)

**2. Plans before:**

```sql
explain analyze select * from time_entries where invoice_id = 1;
explain analyze select project_id, sum(amount_cents) from invoice_lines where project_id in (1, 2) group by project_id;
explain analyze select * from invoices where client_id = 1;
```

The first shows `Seq Scan on time_entries … Filter: (invoice_id = 1) … Rows Removed by Filter: …`.

**3. Migration** (next free number):

```sql
-- src/main/resources/db/migration/V11__add_foreign_key_indexes.sql
create index time_entries_invoice_id_index on time_entries (invoice_id);
create index invoice_lines_project_id_index on invoice_lines (project_id);
create index invoice_lines_invoice_id_index on invoice_lines (invoice_id);
create index invoices_client_id_index on invoices (client_id);
```

(`V11` if you have `V1`–`V10` from class.) Each index is justified by one of the plans above. The
`invoice_lines_invoice_id_index` line is **optional**: the invoice page loads lines by `invoice_id`;
keep it only if `explain analyze select * from invoice_lines where invoice_id = 1;` shows a
`Seq Scan` that matters on your data, and write down the plan either way.

Do **not** add one for `projects.client_id`: lesson 03 already created `projects_client_id_index`.

**4. Plans after:** the first now shows `Index Scan using time_entries_invoice_id_index`. The two small
tables may still show `Seq Scan` — PostgreSQL correctly prefers it for a few rows; note that. Write the
before/after lines and one sentence per index into `docs/performance.md`.
</details>

<details>
<summary>Task 3 — Query count for the paginated invoices page (Intermediate)</summary>

**1. Paginate.** In `InvoiceRepository`, replace `findAllByOrderByIssuedOnDescIdDesc()`:

```java
// src/main/java/ee/ta25/billable/invoice/InvoiceRepository.java (replace the list method)
// imports: org.springframework.data.domain.Page, org.springframework.data.domain.Pageable
  @Override
  @EntityGraph(attributePaths = "client")
  Page<Invoice> findAll(Pageable pageable);
```

`InvoiceService.findAll()` becomes `Page<Invoice> findPage(Pageable pageable)` returning
`invoices.findAll(pageable)`; the controller takes
`@PageableDefault(size = 20, sort = {"issuedOn", "id"}, direction = Sort.Direction.DESC) Pageable pageable`
(as the clients list in lesson 04) and the template loops over `${invoices.content}` with the same
pagination links as `clients/index.html` (`current = ${invoices.number + 1}`, because the URL's `?page=1`
is 1-based since lesson 04 while `getNumber()` is 0-based). `@EntityGraph("client")` is a `@ManyToOne`, so the join does
not break `LIMIT` — never put `lines` here (README, concept 8).

**2. Test.**

```java
// src/test/java/ee/ta25/billable/invoice/InvoicePageQueryCountTest.java
package ee.ta25.billable.invoice;

import static org.assertj.core.api.Assertions.assertThat;

import ee.ta25.billable.TestDatabase;
import ee.ta25.billable.TestcontainersConfiguration;
import jakarta.persistence.EntityManager;
import jakarta.persistence.EntityManagerFactory;
import org.hibernate.SessionFactory;
import org.hibernate.stat.Statistics;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest;
import org.springframework.context.annotation.Import;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;
import org.springframework.jdbc.core.JdbcTemplate;

@DataJpaTest(properties = "spring.jpa.properties.hibernate.generate_statistics=true")
@Import(TestcontainersConfiguration.class)
class InvoicePageQueryCountTest {

  @Autowired private JdbcTemplate jdbc;
  @Autowired private EntityManager em;
  @Autowired private EntityManagerFactory emf;
  @Autowired private InvoiceRepository invoices;

  @Test
  void the_first_page_needs_the_same_two_statements_for_30_or_60_invoices() {
    TestDatabase db = new TestDatabase(jdbc);
    createInvoices(db, 1, 30);
    long thirty = statementsForFirstPage();

    createInvoices(db, 31, 60);
    long sixty = statementsForFirstPage();

    assertThat(sixty).isEqualTo(thirty).isLessThanOrEqualTo(2);
  }

  private void createInvoices(TestDatabase db, int from, int to) {
    for (int i = from; i <= to; i++) {
      db.invoice(db.client("Client " + i), "2026-%04d".formatted(i), "draft");
    }
  }

  private long statementsForFirstPage() {
    em.clear();
    Statistics statistics = emf.unwrap(SessionFactory.class).getStatistics();
    statistics.clear();
    invoices
        .findAll(PageRequest.of(0, 20, Sort.by(Sort.Direction.DESC, "issuedOn", "id")))
        .forEach(invoice -> invoice.getClient().getName());
    return statistics.getPrepareStatementCount();
  }
}
```

Two statements: the page (with the clients joined) and the `count`. Remove `@EntityGraph` and the count
jumps to 22.
</details>

<details>
<summary>Task 4 — A security fix with a test (Intermediate)</summary>

Example: `clients/show.html` printed the name with `th:utext="${client.name}"` "so the ampersand shows
correctly". That is stored XSS. Test first:

```java
// src/test/java/ee/ta25/billable/client/ClientXssTest.java
package ee.ta25.billable.client;

import static org.assertj.core.api.Assertions.assertThat;

import ee.ta25.billable.IntegrationTest;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.web.servlet.assertj.MockMvcTester;

class ClientXssTest extends IntegrationTest {

  @Autowired private MockMvcTester mvc;

  @Test
  void client_names_are_escaped_on_the_detail_page() {
    long id = db.client("<script>alert('x')</script>");

    assertThat(mvc.get().uri("/clients/{id}", id))
        .hasStatusOk()
        .bodyText()
        .doesNotContain("<script>alert('x')</script>")
        .contains("&lt;script&gt;");
  }
}
```

It fails with `th:utext` and passes after changing it to `th:text`. Commit as
`fix(security): escape client name on detail page (stored XSS)`.

Another common find: a controller method taking `@ModelAttribute TimeEntry entry` (the **entity**)
instead of a form object — then a request can set any field, including `invoice`. The fix is a form
class with only the allowed fields; the test posts an extra `invoice=1` parameter and asserts the saved
entry's `invoice_id` is `null`.
</details>

<details>
<summary>Task 5 — Concurrency evidence (Advanced)</summary>

1. Put the lesson 13 version of the generator back temporarily:
   `git stash` your working changes, or `git checkout <lesson-13-commit> -- src/main/java/ee/ta25/billable/invoice/InvoiceNumberGenerator.java`
   (and restore `findMaxNumberLike` if you removed it).
2. Run `./mvnw test -Dtest=InvoiceNumberingConcurrencyTest` three times. Copy the failure output
   (the list of `DataIntegrityViolationException`s with `invoices_number_key`).
3. Restore the new version (`git checkout HEAD -- …` / `git stash pop`) and run it three times again:
   green every time.
4. Optional, stronger: raise `INVOICES` to 50 and set
   `@SpringBootTest(properties = "spring.datasource.hikari.maximum-pool-size=50")` on a copy of the test
   class — more threads, more pressure, still gap-free.
5. In `docs/algorithm.md`, paste both outputs and explain in two sentences, e.g.: *"With max + 1, several
   transactions read the same maximum under READ COMMITTED; the unique constraint rejected the later
   inserts. With the locked sequence row, each transaction waited for the previous commit and continued
   from its value, so ten parallel requests produced 2026-0001 to 2026-0010."*
</details>
