# 04 — MVC: controllers and views · Spring Boot

[← Concepts](README.md) · Starting point: end of lesson 03 · Estimated time in class: 2 h · Independent work tasks: [README](README.md#independent-work-2-h)

## What we build today

- `common.NotFoundException` — an exception that becomes a 404 response
- a grouped count query and a small projection interface `project.ClientProjectCount`
- `client.ClientController` with `index` (paginated, with project counts) and `show` (client + projects)
- page numbers in the URL from 1 (`?page=1`), like Laravel: `spring.data.web.pageable.one-indexed-parameters=true`
- `common.Formatters` — a bean with `euros(long cents)` that formats `150000` as `1 500,00 €` with `NumberFormat`
- a "Clients" link in `layout.html`; every new page uses the layout from lesson 01, so it gets the
  flash box for free
- templates `clients/index.html`, `clients/show.html` and `error/404.html`

**Final result:** `/clients` shows a paginated table of clients; clicking a name opens `/clients/{id}` with
the client's details and projects; `/clients/999` shows a styled "Not found" page with status 404.

Start the application with the `dev` profile as in lesson 03:

```bash
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

If you still have `QueryPlayground` from lesson 03, delete it now.

---

## Step 1 — `NotFoundException`

**Why:** when a URL contains an id that does not exist, the answer must be **status 404** and a friendly
page. In Spring, the controller does the lookup itself (`findById`), so we need an exception that says
"not found" in a way Spring understands.

```java
// src/main/java/ee/ta25/billable/common/NotFoundException.java
package ee.ta25.billable.common;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

/** Thrown when a requested record does not exist. Spring turns it into a 404 response. */
@ResponseStatus(HttpStatus.NOT_FOUND)
public class NotFoundException extends RuntimeException {

  public NotFoundException(String message) {
    super(message);
  }
}
```

**What happens:**

1. `extends RuntimeException` — an *unchecked* exception, so methods do not need `throws` clauses.
2. `@ResponseStatus(HttpStatus.NOT_FOUND)` — when this exception leaves a controller method, Spring MVC
   sets the response status to 404 and asks Spring Boot's error handling to render an error page.
3. It lives in `common`, because every feature (clients, projects, invoices) needs it.

Spring also has a built-in `ResponseStatusException(HttpStatus.NOT_FOUND, "...")` that you can throw
directly. Our own class is shorter to write in every controller, and its name says exactly what happened.

---

## Step 2 — Counting projects for a page of clients

**Why:** the list shows how many projects each client has. The naive way — calling
`projects.countByClientId(id)` for every row — runs one query per client: 1 + 10 queries for a page of 10.
That is the N+1 problem (lesson 06). Instead we ask **once** for all clients on the page.

A projection interface describes the shape of one result row:

```java
// src/main/java/ee/ta25/billable/project/ClientProjectCount.java
package ee.ta25.billable.project;

/** One row of {@link ProjectRepository#countByClientIds}: a client id and its number of projects. */
public interface ClientProjectCount {

  Long getClientId();

  long getProjectCount();
}
```

Add the query method to `ProjectRepository`:

```java
// src/main/java/ee/ta25/billable/project/ProjectRepository.java
package ee.ta25.billable.project;

import java.util.Collection;
import java.util.List;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

public interface ProjectRepository extends JpaRepository<Project, Long> {

  long countByClientId(Long clientId);

  List<Project> findByClientIdOrderByName(Long clientId);

  @Query(
      """
      select p.client.id as clientId, count(p) as projectCount
      from Project p
      where p.client.id in :clientIds
      group by p.client.id
      """)
  List<ClientProjectCount> countByClientIds(@Param("clientIds") Collection<Long> clientIds);

  // ...keep the methods you added in the lesson 03 independent work
}
```

**What happens:**

1. `@Query` gives the query explicitly in **JPQL** — a query language that uses entity and field names
   (`Project`, `p.client.id`), not table and column names. Hibernate translates it to SQL.
2. The text block (`"""`) keeps a multi-line query readable.
3. `as clientId` and `as projectCount` are aliases. Spring Data matches them to the getters of
   `ClientProjectCount` (`getClientId()`, `getProjectCount()`) and creates an object for every row.
4. `in :clientIds` with a `Collection<Long>` — Hibernate expands it to `in (?, ?, ?, ...)`, one bound
   parameter per id.
5. The generated SQL is one grouped query:
   `select p1_0.client_id, count(p1_0.id) from projects p1_0 where p1_0.client_id in (?,?,?) group by p1_0.client_id`.

A client with **no** projects does not appear in the result at all (there is nothing to group). The
controller handles that by starting every client at 0.

---

## Step 3 — `ClientController`

```java
// src/main/java/ee/ta25/billable/client/ClientController.java
package ee.ta25.billable.client;

import ee.ta25.billable.common.NotFoundException;
import ee.ta25.billable.project.ClientProjectCount;
import ee.ta25.billable.project.ProjectRepository;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;

@Controller
@RequestMapping("/clients")
public class ClientController {

  private final ClientRepository clients;
  private final ProjectRepository projects;

  ClientController(ClientRepository clients, ProjectRepository projects) {
    this.clients = clients;
    this.projects = projects;
  }

  @GetMapping
  public String index(
      @PageableDefault(size = 10, sort = {"name", "id"}) Pageable pageable, Model model) {
    Page<Client> page = clients.findAll(pageable);
    model.addAttribute("page", page);
    model.addAttribute("projectCounts", projectCountsFor(page.getContent()));
    return "clients/index";
  }

  @GetMapping("/{id}")
  public String show(@PathVariable Long id, Model model) {
    Client client =
        clients.findById(id).orElseThrow(() -> new NotFoundException("Client " + id + " not found"));
    model.addAttribute("client", client);
    model.addAttribute("projects", projects.findByClientIdOrderByName(id));
    return "clients/show";
  }

  /** Number of projects per client id, for one page of clients, in a single query. */
  private Map<Long, Long> projectCountsFor(List<Client> pageOfClients) {
    Map<Long, Long> counts = new HashMap<>();
    if (pageOfClients.isEmpty()) {
      return counts;
    }
    List<Long> ids = pageOfClients.stream().map(Client::getId).toList();
    ids.forEach(id -> counts.put(id, 0L));
    for (ClientProjectCount row : projects.countByClientIds(ids)) {
      counts.put(row.getClientId(), row.getProjectCount());
    }
    return counts;
  }
}
```

**What happens in `index`:**

1. `@Controller` (not `@RestController`) — the methods return a **view name**, and Thymeleaf renders it.
2. `@RequestMapping("/clients")` on the class is a prefix for every method: `@GetMapping` alone means
   `GET /clients`, `@GetMapping("/{id}")` means `GET /clients/{id}`.
3. **`Pageable`** — Spring reads `?page=`, `?size=` and `?sort=` from the URL and builds a `Pageable`
   object. `@PageableDefault(size = 10, sort = {"name", "id"})` is used when the URL has none: first page,
   10 rows, sorted by name, then by id as a tie-breaker. By default Spring counts URL pages from 0
   (`?page=0` is the first page); Laravel counts from 1. We switch Spring to 1 below, so both tracks
   have the same URLs.
4. `clients.findAll(pageable)` returns a `Page<Client>`: the clients of this page **plus** the total count.
   It runs two queries: `select ... from clients order by name, id offset ? rows fetch first ? rows only`
   and `select count(*) from clients`.
5. `projectCountsFor(...)` runs **one** extra query for the whole page and returns a map
   `client id → number of projects`. Every id starts at 0, so clients without projects show 0. (This helper
   is data preparation for the view; in lesson 07 such code moves into a service.)
   Laravel's `withCount` puts the count into the **same** query as a sub-query. Hibernate can do that too:
   `@Formula("(select count(*) from projects p where p.client_id = id)")` on a field of `Client`. We do not
   use it, because a `@Formula` runs on **every** load of a client, also where nobody needs the count.
6. The template gets two attributes: `page` and `projectCounts`.

**What happens in `show`:**

1. `@PathVariable Long id` takes `{id}` from the URL and converts it to a `Long`. `/clients/abc` cannot be
   converted, so Spring answers **400 Bad Request** before the method runs.
2. `findById(id)` returns `Optional<Client>`. `.orElseThrow(...)` gives the client, or throws our
   `NotFoundException` → 404.
3. The projects are loaded **in the controller** with `findByClientIdOrderByName(id)`. We do not use
   `client.getProjects()` in the view: with `open-in-view=false` (lesson 03), the session is closed by the
   time the template renders, and touching the lazy list would throw `LazyInitializationException`. That
   is the framework enforcing "views do not query".

**Page numbers from 1.** Add to `application.properties`:

```properties
# src/main/resources/application.properties
spring.data.web.pageable.one-indexed-parameters=true
```

Now `?page=1` is the first page, as in Laravel. Spring subtracts 1 when it builds the `Pageable` (a
missing, `0` or negative `page` also gives the first page). **Inside Java nothing changes**:
`Pageable.getPageNumber()` and `Page.getNumber()` (`page.number` in the template) still count from 0.
Only the number in the URL is 1-based, so a template adds 1 when it writes a page number into a link or
shows it to a person (Step 6).

**Safety note:** `Pageable` accepts `?size=` from the user. Someone could ask for `?size=100000`. Spring
limits it to 2000 by default; set a stricter limit in `application.properties`:

```properties
# src/main/resources/application.properties
spring.data.web.pageable.max-page-size=50
```

---

## Step 4 — `Formatters`: euros for templates

**Why:** amounts are stored as integer cents, but the page must show `1 500,00 €`. We do not write the
formatting rules ourselves: Java's `NumberFormat` already knows how Estonian writes money (comma for
decimals, space between thousands, `€` after the number). The class is a Spring bean, so Thymeleaf can
call it with `${@formatters...}`. Lesson 10 replaces it with `common.Money`.

```java
// src/main/java/ee/ta25/billable/common/Formatters.java
package ee.ta25.billable.common;

import java.math.BigDecimal;
import java.text.NumberFormat;
import java.util.Locale;
import org.springframework.stereotype.Component;

/**
 * Display helpers for templates, e.g. {@code ${@formatters.euros(6500)}}. Temporary: lesson 10
 * replaces the money formatting with {@code Money}.
 */
@Component
public class Formatters {

  private static final Locale ESTONIAN = Locale.of("et", "EE");

  /** Formats integer cents as euros: 150000 → "1 500,00 €", -5 → "−0,05 €". */
  public String euros(long cents) {
    return NumberFormat.getCurrencyInstance(ESTONIAN).format(BigDecimal.valueOf(cents, 2));
  }

  /** Like {@link #euros(long)}, but shows "—" when there is no amount (a nullable column). */
  public String eurosOrDash(Long cents) {
    return cents == null ? "—" : euros(cents);
  }
}
```

**What happens:**

1. `@Component` registers the class as a bean. Its bean name is the class name with a small first letter:
   `formatters`. In Thymeleaf, `@formatters` means "the bean named `formatters`".
2. `BigDecimal.valueOf(150000, 2)` means "150000 with the decimal point moved 2 places": exactly
   `1500.00`. No `double` is involved, so rule R1 (no floats for money) holds. (For display only, even a
   `double` would be fine: R1 is about storing and calculating, and the formatter rounds to 2 decimals.)
3. `NumberFormat.getCurrencyInstance(Locale.of("et", "EE"))` returns a formatter for euros in Estonia.
   It prints `1 500,00 €`. A negative amount gets a real minus sign (U+2212): `−5,00 €`.
4. The "spaces" in the output are **non-breaking spaces** (U+00A0), not normal spaces. The browser shows
   them the same, but it never breaks the line between `1 500,00` and `€`. In a test, compare with
   `"1\u00a0500,00\u00a0€"`, not with a string you typed with the space bar.
5. We create a new `NumberFormat` on every call: a `NumberFormat` object is **not thread-safe**, and a
   bean is shared by all requests. Creating one is cheap.
6. `eurosOrDash(Long)` exists because the money columns in `Project` are nullable `Long`s. A fixed
   project has no hourly rate, and the page should show `—`, not `0,00 €`. The two methods have
   **different names** on purpose: overloads with `long` and `Long` can confuse Thymeleaf's method
   lookup when the value is `null`.

You can try the formatter without starting the application, in `jshell`:

```text
jshell> java.text.NumberFormat.getCurrencyInstance(Locale.of("et", "EE")).format(java.math.BigDecimal.valueOf(150000, 2))
$1 ==> "1 500,00 €"
```

**Check it works:** we see it on the client page in step 7. (In lesson 11 you will write unit tests for
exactly this kind of method.)

---

## Step 5 — A "Clients" link in the layout

Open `src/main/resources/templates/layout.html` from lesson 01. It is one fragment,
`layout(title, content)`, with the head, the navigation, the flash box and the footer. Find the second
`<ul>` inside `<nav>` and add a Clients link (keep your Home and About links):

```html
<!-- src/main/resources/templates/layout.html — the second <ul> inside <nav> -->
<ul>
  <li><a th:href="@{/}">Home</a></li>
  <li><a th:href="@{/clients}">Clients</a></li>
  <li><a th:href="@{/about}">About</a></li>
</ul>
```

Look at the flash box that is already in the file, between `</header>` and `<main>`:

```html
<!-- src/main/resources/templates/layout.html — already there since lesson 01 -->
<div class="container" th:if="${status}">
  <article role="status" th:text="${status}">Saved.</article>
</div>
```

**What happens:**

1. `layout.html` is never shown as a page itself. Pages wrap themselves in it with
   `th:replace="~{layout :: layout(~{::title}, ~{::main})}"`.
2. `th:href="@{/clients}"` builds the link. `@{...}` adds the application's context path if it ever runs
   under a sub-path (e.g. `/billable/clients`), which a hard-coded `href="/clients"` would not.
3. The flash box is our place for one-time messages. The layout is rendered with the page's model, so
   `th:if="${status}"` sees the page's attributes. If there is no `status` attribute, **nothing** is
   rendered. Because it is in the layout, every page shows it without doing anything — ready for
   lesson 05, where a controller calls `redirectAttributes.addFlashAttribute("status", "Client saved.")`
   before a redirect. Flash attributes survive exactly one redirect and then disappear.

---

## Step 6 — The client list template

```html
<!-- src/main/resources/templates/clients/index.html -->
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      th:replace="~{layout :: layout(~{::title}, ~{::main})}">
<head>
  <title>Clients · Billable</title>
</head>
<body>
<main class="container">
  <h1>Clients</h1>

  <p th:if="${page.empty}">No clients yet.</p>

  <th:block th:unless="${page.empty}">
    <p th:text="|${page.totalElements} clients|">3 clients</p>

    <table>
      <thead>
        <tr>
          <th scope="col">Name</th>
          <th scope="col">Email</th>
          <th scope="col">VAT number</th>
          <th scope="col">Projects</th>
        </tr>
      </thead>
      <tbody>
        <tr th:each="client : ${page.content}">
          <td>
            <a th:href="@{/clients/{id}(id=${client.id})}" th:text="${client.name}">Acme OÜ</a>
          </td>
          <td th:text="${client.email}">billing@acme.test</td>
          <td th:text="${client.vatNumber} ?: '—'">EE101234567</td>
          <td th:text="${projectCounts.get(client.id)}">3</td>
        </tr>
      </tbody>
    </table>

    <nav th:if="${page.totalPages > 1}" th:with="current=${page.number + 1}" aria-label="Pagination">
      <ul>
        <li>
          <a th:if="${page.hasPrevious()}" th:href="@{/clients(page=${current - 1})}"
             rel="prev">← Previous</a>
          <span th:unless="${page.hasPrevious()}" aria-disabled="true">← Previous</span>
        </li>
      </ul>
      <ul>
        <li th:text="|Page ${current} of ${page.totalPages}|">Page 1 of 3</li>
      </ul>
      <ul>
        <li>
          <a th:if="${page.hasNext()}" th:href="@{/clients(page=${current + 1})}"
             rel="next">Next →</a>
          <span th:unless="${page.hasNext()}" aria-disabled="true">Next →</span>
        </li>
      </ul>
    </nav>
  </th:block>
</main>
</body>
</html>
```

**What happens:**

1. `th:replace="~{layout :: layout(~{::title}, ~{::main})}"` wraps this page in the layout from
   lesson 01: the page's `<title>` and `<main>` go into the layout, which adds head, nav, flash box and
   footer.
2. `${page.empty}` calls `Page.isEmpty()`. With no clients, the page shows the **empty state** and hides
   the table (`th:block` is an invisible wrapper element that Thymeleaf removes from the output).
3. `th:each="client : ${page.content}"` repeats the `<tr>` for every client on this page.
4. `th:text` **escapes** the value: a name containing `<script>` is shown as text (step 9 proves it). The
   text inside the tag (`Acme OÜ`) is only a placeholder for opening the file without the server.
5. `@{/clients/{id}(id=${client.id})}` fills the `{id}` path variable: `/clients/1`.
6. `${client.vatNumber} ?: '—'` — the "Elvis" operator: if the value is `null`, use `—`.
7. `${projectCounts.get(client.id)}` reads from the map the controller built — no query in the view.
8. Pagination: `page.number` is **0-based** inside Java, but our URLs are 1-based (Step 3). So
   `th:with="current=${page.number + 1}"` defines a local variable `current`, the 1-based number of this
   page, for everything inside the `<nav>`. "Page 2 of 3" shows `current`; the "Previous" link points to
   `page=${current - 1}` and "Next" to `page=${current + 1}`. `@{/clients(page=...)}` adds it as a query
   parameter.
   `hasPrevious()` / `hasNext()` decide whether a link is shown. With 3 clients there is one page, so the
   whole `<nav>` is hidden.

**Check it works:** open `http://localhost:8080/clients`. You see Acme OÜ (3 projects), Birch & Daughters
(2), Kuressaare Kala AS (2). In the log you see three queries: the page, the count, and the grouped project
count.

To see pagination, add 25 extra clients to your **local** database. This inserts test *data*; it does not
change the schema, so it is fine on your own machine:

```bash
docker compose exec -T postgres psql -U billable -d billable <<'SQL'
INSERT INTO clients (name, email, created_at, updated_at)
SELECT 'Test client ' || lpad(n::text, 2, '0'), 'test' || n || '@example.test', now(), now()
FROM generate_series(1, 25) AS n;
SQL
```

Reload `/clients`: "Page 1 of 3", and "Next →" goes to `?page=2` (the second page, as in Laravel).
`/clients?page=3` shows the last page and "Page 3 of 3". (To return to the standard data:
`docker compose down -v` and start the application again with the `dev` profile.)

**Commit:** `feat(clients): paginated clients list`

---

## Step 7 — The client detail template

```html
<!-- src/main/resources/templates/clients/show.html -->
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      th:replace="~{layout :: layout(~{::title}, ~{::main})}">
<head>
  <title th:text="|${client.name} · Billable|">Client · Billable</title>
</head>
<body>
<main class="container">
  <p><a th:href="@{/clients}">← All clients</a></p>

  <h1 th:text="${client.name}">Acme OÜ</h1>

  <table>
    <tbody>
      <tr>
        <th scope="row">Email</th>
        <td><a th:href="|mailto:${client.email}|" th:text="${client.email}">billing@acme.test</a></td>
      </tr>
      <tr>
        <th scope="row">VAT number</th>
        <td th:text="${client.vatNumber} ?: '—'">EE101234567</td>
      </tr>
      <tr>
        <th scope="row">Client since</th>
        <td th:text="${#temporals.format(client.createdAt, 'dd.MM.yyyy')}">30.09.2026</td>
      </tr>
    </tbody>
  </table>

  <h2>Projects</h2>

  <p th:if="${projects.isEmpty()}">This client has no projects yet.</p>

  <table th:unless="${projects.isEmpty()}">
    <thead>
      <tr>
        <th scope="col">Name</th>
        <th scope="col">Billing</th>
        <th scope="col">Hourly rate</th>
        <th scope="col">Fixed price</th>
        <th scope="col">Budget cap</th>
        <th scope="col">Rounding</th>
        <th scope="col">Status</th>
      </tr>
    </thead>
    <tbody>
      <tr th:each="project : ${projects}">
        <td th:text="${project.name}">Website redesign</td>
        <td th:text="${project.billingType.label()}">Hourly</td>
        <td th:text="${@formatters.eurosOrDash(project.hourlyRateCents)}">65,00 €</td>
        <td th:text="${@formatters.eurosOrDash(project.fixedPriceCents)}">—</td>
        <td th:text="${@formatters.eurosOrDash(project.budgetCapCents)}">—</td>
        <td th:text="|${project.roundingMinutes} min|">15 min</td>
        <td th:text="${project.archived} ? 'Archived' : 'Active'">Active</td>
      </tr>
    </tbody>
  </table>
</main>
</body>
</html>
```

**What happens:**

1. `<title th:text="|${client.name} · Billable|">` is a dynamic title: the browser tab shows
   "Acme OÜ · Billable" (`|...|` is explained in point 2). The page's `<title>` is filled from the
   page's model, then the layout puts it into its `<head>`.
2. `|mailto:${client.email}|` is a *literal substitution*: text with expressions inside `|...|`.
3. `#temporals.format(client.createdAt, 'dd.MM.yyyy')` formats the `LocalDateTime` in the Estonian style.
   `#temporals` is built into Thymeleaf 3.1.
4. `project.billingType.label()` calls the enum's `label()` method. (It is called `label()`, not
   `getLabel()`, so we write the parentheses.)
5. `${@formatters.eurosOrDash(project.hourlyRateCents)}` calls the `Formatters` bean from step 4.
6. `${project.archived}` calls `isArchived()` — Thymeleaf finds boolean `is...` getters by property name.
   The rule "archived means `archivedAt` is not null" stays in the entity.
7. `project.billingType`, `project.name` etc. are plain fields that were loaded with the project, so no
   lazy loading happens. We never touch `project.client` here.

**Check it works:** click Acme OÜ. You see `EE101234567` and three projects. "Support retainer" shows
`Capped hourly`, `60,00 €`, `—`, `2 000,00 €`, `6 min`, `Active`; "Old intranet" shows `Archived`.

**Commit:** `feat(clients): client detail page with projects`

---

## Step 8 — A custom 404 page

Open `/clients/999`. The status is already 404 (our exception), but you see Spring Boot's plain
"Whitelabel Error Page". Spring Boot looks for a template named after the status code in the `error`
folder:

```html
<!-- src/main/resources/templates/error/404.html -->
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      th:replace="~{layout :: layout(~{::title}, ~{::main})}">
<head>
  <title>Not found · Billable</title>
</head>
<body>
<main class="container">
  <h1>Not found</h1>
  <p>The page or record you asked for does not exist. It may have been deleted, or the link is wrong.</p>
  <p><small th:text="${path}">/clients/999</small></p>
  <p><a th:href="@{/}">Back to the home page</a></p>
</main>
</body>
</html>
```

The error page uses the same layout, and that brings one surprise. Spring Boot's error model has an
attribute called `status` too — the HTTP code, `404` — so the layout's flash box would show a box with
"404" at the top. Change the flash box in `layout.html` so that it ignores error pages:

```html
<!-- src/main/resources/templates/layout.html — replace the flash box -->
<div class="container" th:if="${status != null and timestamp == null}">
  <article role="status" th:text="${status}">Saved.</article>
</div>
```

**What happens:**

1. `NotFoundException` leaves `show(...)`. Because of `@ResponseStatus(NOT_FOUND)`, Spring MVC calls
   `response.sendError(404, ...)`.
2. The servlet container forwards the request to Spring Boot's `/error` endpoint (`BasicErrorController`).
3. `BasicErrorController` looks for a view `error/404` (exact status), then `error/4xx` (any 4xx), then
   falls back to the Whitelabel page. It finds our template and renders it **with status 404**.
4. The error model contains `status`, `error`, `path` and `timestamp`. We show the `path` so the user sees
   which URL failed. We do not show exception messages or stack traces to users.
5. `status` and `error` are also the names of our flash messages (lesson 05 adds `error`). Only the error
   model has a `timestamp`, so `timestamp == null` means "a normal page": there, `status` is a flash
   message. On the error page the flash box is skipped.
6. The same page is used for unknown URLs like `/nothing-here`.

**Check it works:**

```bash
curl -i http://localhost:8080/clients/999
```

The first line is `HTTP/1.1 404`, and the body contains "Not found" inside your layout, with no "404"
box above it. `/clients/abc`
gives **400** (not a number) — Spring's default error page for 400, which is fine for now.

**Commit:** `feat(errors): custom 404 page`

---

## Step 9 — Prove that output is escaped

**Why:** XSS is one of the most common web vulnerabilities. Let us see `th:text` protect us.

Insert a client with a "dangerous" name into your local database:

```bash
docker compose exec -T postgres psql -U billable -d billable <<'SQL'
INSERT INTO clients (name, email, created_at, updated_at)
VALUES ('<script>alert(''hi'')</script>', 'xss@example.test', now(), now());
SQL
```

Open `/clients`. The name is shown as text; no alert appears. View the page source: it contains
`&lt;script&gt;alert(&#39;hi&#39;)&lt;/script&gt;`.

Now, **only as an experiment**, change `th:text="${client.name}"` to `th:utext="${client.name}"` in
`clients/index.html` and reload (DevTools reloads templates automatically). The alert appears — the browser
ran the "name" as JavaScript. **Change it back to `th:text`.**

Reset your data (`docker compose down -v`, start again with the `dev` profile), then:

```bash
./mvnw spotless:apply
./mvnw verify
```

---

## Troubleshooting

| Error / symptom | Cause | Fix |
|---|---|---|
| `TemplateInputException: Error resolving template [clients/index]` | File missing or in the wrong folder | It must be `src/main/resources/templates/clients/index.html` |
| `LazyInitializationException: ... could not initialize proxy - no Session` while rendering | The template touched a lazy relation (`client.projects`, `project.client.name`) | Load the data in the controller with a repository query and pass it to the view |
| `EL1004E: Method call: Method euro(java.lang.Long) cannot be found on type ...Formatters` | Typo in the method name, or wrong argument type | Check the name (`eurosOrDash`); pass the entity field (a `Long`) |
| `EL1008E: Property or field 'label' cannot be found on object of type 'BillingType'` | `${project.billingType.label}` without `()` | The method is `label()`, not a getter: write `label()` |
| `EL1007E: Property or field 'name' cannot be found on null` | An attribute is missing from the model (typo in `addAttribute`) | Names in `model.addAttribute("...")` and `${...}` must match |
| Whitelabel Error Page instead of your 404 | Template not at `templates/error/404.html` | Check folder name `error` (singular) |
| `/clients/999` returns **500** | You threw a plain `RuntimeException`, or `.get()` on the `Optional` | Use `orElseThrow(() -> new NotFoundException(...))` |
| "Next →" skips a page or stays on the same page; `?page=1` shows the second page | `one-indexed-parameters=true` is missing, or a link uses `page.number` without `+ 1` | Add the property (Step 3); build links from `current = page.number + 1` (Step 6) |
| `No property 'foo' found for type 'Client'` → 500 for `/clients?sort=foo` | `Pageable` accepts any `sort` from the URL | The advanced task replaces this with a whitelist |
| `Parameter 0 of constructor in ...ClientController required a bean of type 'ProjectRepository'` | Repository in a package outside `ee.ta25.billable` | Keep every class under the main application's package |
| Page shows the placeholder text (`Acme OÜ`) for every row | You opened the HTML file directly, not through the server | Use `http://localhost:8080/clients` |

---

## Recap

- `common.NotFoundException` — `@ResponseStatus(NOT_FOUND)`, used with `findById(...).orElseThrow(...)`.
- `project.ClientProjectCount` + `ProjectRepository.countByClientIds` — one grouped query for a page of
  clients instead of one query per row.
- `client.ClientController` — thin `index` (`Pageable`, `Page<Client>`) and `show` (`@PathVariable`).
- `application.properties` — `one-indexed-parameters=true` (URL `?page=1` is the first page;
  `page.number` stays 0-based in Java and templates) and `max-page-size=50`.
- `common.Formatters` — `euros(long)` with `NumberFormat` and `BigDecimal.valueOf(cents, 2)`; temporary until lesson 10.
- `templates/layout.html` — Clients link; the flash box skips error pages (`timestamp == null`).
  Every page wraps itself in the layout with `th:replace="~{layout :: layout(~{::title}, ~{::main})}"`.
- `templates/clients/index.html`, `clients/show.html` — display only, `th:text` everywhere, empty states.
- `templates/error/404.html` — custom 404, rendered by Spring Boot with the correct status.

---

## Independent work — solutions

> **The tasks are not here.** The task descriptions (Basic / Intermediate / Advanced, with acceptance
> criteria) are in this lesson's [README.md → Independent work (~2 h)](README.md#independent-work-2-h).
> Read them there first and build the features in your project. This section only contains the
> solutions — try each task yourself, then open a solution to compare or when you are stuck.

<details>
<summary><strong>Basic</strong> — <code>ProjectController</code> with shallow nesting</summary>

The project page shows the client's name, so the client must be loaded **before** the view renders
(`open-in-view=false`). A JPQL `join fetch` loads the project and its client in one query. Add to
`ProjectRepository`:

```java
// src/main/java/ee/ta25/billable/project/ProjectRepository.java — add inside the interface
  @Query("select p from Project p join fetch p.client where p.id = :id")
  Optional<Project> findWithClientById(@Param("id") Long id);
```

(Import `java.util.Optional`.) `join fetch` means "join the clients table and fill `project.client` with
the real object, not a lazy placeholder". Lesson 06 explains fetching in depth.

```java
// src/main/java/ee/ta25/billable/project/ProjectController.java
package ee.ta25.billable.project;

import ee.ta25.billable.client.Client;
import ee.ta25.billable.client.ClientRepository;
import ee.ta25.billable.common.NotFoundException;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

@Controller
public class ProjectController {

  private final ClientRepository clients;
  private final ProjectRepository projects;

  ProjectController(ClientRepository clients, ProjectRepository projects) {
    this.clients = clients;
    this.projects = projects;
  }

  @GetMapping("/clients/{clientId}/projects")
  public String index(@PathVariable Long clientId, Model model) {
    Client client =
        clients
            .findById(clientId)
            .orElseThrow(() -> new NotFoundException("Client " + clientId + " not found"));
    model.addAttribute("client", client);
    model.addAttribute("projects", projects.findByClientIdOrderByName(clientId));
    return "projects/index";
  }

  @GetMapping("/projects/{id}")
  public String show(@PathVariable Long id, Model model) {
    Project project =
        projects
            .findWithClientById(id)
            .orElseThrow(() -> new NotFoundException("Project " + id + " not found"));
    model.addAttribute("project", project);
    return "projects/show";
  }
}
```

No class-level `@RequestMapping`, because the two URLs have different prefixes — that is shallow nesting:
the list lives under the client, a single project has its own URL.

```html
<!-- src/main/resources/templates/projects/index.html -->
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      th:replace="~{layout :: layout(~{::title}, ~{::main})}">
<head>
  <title th:text="|Projects of ${client.name} · Billable|">Projects · Billable</title>
</head>
<body>
<main class="container">
  <p><a th:href="@{/clients/{id}(id=${client.id})}" th:text="|← ${client.name}|">← Acme OÜ</a></p>

  <h1 th:text="|Projects of ${client.name}|">Projects of Acme OÜ</h1>

  <p th:if="${projects.isEmpty()}">This client has no projects yet.</p>

  <table th:unless="${projects.isEmpty()}">
    <thead>
      <tr>
        <th scope="col">Name</th>
        <th scope="col">Billing</th>
        <th scope="col">Hourly rate</th>
        <th scope="col">Fixed price</th>
        <th scope="col">Budget cap</th>
        <th scope="col">Rounding</th>
        <th scope="col">Status</th>
      </tr>
    </thead>
    <tbody>
      <tr th:each="project : ${projects}">
        <td>
          <a th:href="@{/projects/{id}(id=${project.id})}" th:text="${project.name}">Website redesign</a>
        </td>
        <td th:text="${project.billingType.label()}">Hourly</td>
        <td th:text="${@formatters.eurosOrDash(project.hourlyRateCents)}">65,00 €</td>
        <td th:text="${@formatters.eurosOrDash(project.fixedPriceCents)}">—</td>
        <td th:text="${@formatters.eurosOrDash(project.budgetCapCents)}">—</td>
        <td th:text="|${project.roundingMinutes} min|">15 min</td>
        <td th:text="${project.archived} ? 'Archived' : 'Active'">Active</td>
      </tr>
    </tbody>
  </table>
</main>
</body>
</html>
```

```html
<!-- src/main/resources/templates/projects/show.html -->
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      th:replace="~{layout :: layout(~{::title}, ~{::main})}">
<head>
  <title th:text="|${project.name} · Billable|">Project · Billable</title>
</head>
<body>
<main class="container">
  <p>
    <a th:href="@{/clients/{id}/projects(id=${project.client.id})}"
       th:text="|← Projects of ${project.client.name}|">← Projects of Acme OÜ</a>
  </p>

  <h1 th:text="${project.name}">Website redesign</h1>

  <table>
    <tbody>
      <tr>
        <th scope="row">Client</th>
        <td>
          <a th:href="@{/clients/{id}(id=${project.client.id})}"
             th:text="${project.client.name}">Acme OÜ</a>
        </td>
      </tr>
      <tr><th scope="row">Billing</th><td th:text="${project.billingType.label()}">Hourly</td></tr>
      <tr>
        <th scope="row">Hourly rate</th>
        <td th:text="${@formatters.eurosOrDash(project.hourlyRateCents)}">65,00 €</td>
      </tr>
      <tr>
        <th scope="row">Fixed price</th>
        <td th:text="${@formatters.eurosOrDash(project.fixedPriceCents)}">—</td>
      </tr>
      <tr>
        <th scope="row">Budget cap</th>
        <td th:text="${@formatters.eurosOrDash(project.budgetCapCents)}">—</td>
      </tr>
      <tr><th scope="row">Rounding</th><td th:text="|${project.roundingMinutes} min|">15 min</td></tr>
      <tr>
        <th scope="row">Status</th>
        <td th:text="${project.archived} ? 'Archived' : 'Active'">Active</td>
      </tr>
      <tr>
        <th scope="row">Created</th>
        <td th:text="${#temporals.format(project.createdAt, 'dd.MM.yyyy')}">30.09.2026</td>
      </tr>
    </tbody>
  </table>
</main>
</body>
</html>
```

In `clients/show.html`, link to the list and to each project:

```html
<!-- src/main/resources/templates/clients/show.html — replace <h2>Projects</h2> -->
<h2>Projects <small><a th:href="@{/clients/{id}/projects(id=${client.id})}">(open list)</a></small></h2>

<!-- …and replace the first cell of the project row -->
<td><a th:href="@{/projects/{id}(id=${project.id})}" th:text="${project.name}">Website redesign</a></td>
```

The project table now appears twice. To avoid duplication, move it into a fragment
(`templates/projects/table.html` with `th:fragment="table(projects)"`) and include it from both pages with
`th:replace="~{projects/table :: table(${projects})}"`.

Check: `/clients/1/projects` lists 3 projects; `/projects/1` shows Website redesign and a link to Acme OÜ;
the log shows **one** query for `/projects/1` (projects joined with clients); `/clients/999/projects` and
`/projects/999` give 404 (`curl -i`).

</details>

<details>
<summary><strong>Intermediate</strong> — Search by name</summary>

Repository — a paginated version of the lesson 03 search:

```java
// src/main/java/ee/ta25/billable/client/ClientRepository.java — add inside the interface
  Page<Client> findByNameContainingIgnoreCase(String text, Pageable pageable);
```

(Imports: `org.springframework.data.domain.Page`, `org.springframework.data.domain.Pageable`.)

Controller — replace `index(...)` (import `org.springframework.web.bind.annotation.RequestParam`):

```java
// src/main/java/ee/ta25/billable/client/ClientController.java — replace index(...)
  @GetMapping
  public String index(
      @RequestParam(defaultValue = "") String q,
      @PageableDefault(size = 10, sort = {"name", "id"}) Pageable pageable,
      Model model) {
    String search = q.strip();
    Page<Client> page =
        search.isEmpty()
            ? clients.findAll(pageable)
            : clients.findByNameContainingIgnoreCase(search, pageable);
    model.addAttribute("page", page);
    model.addAttribute("projectCounts", projectCountsFor(page.getContent()));
    model.addAttribute("q", search);
    return "clients/index";
  }
```

1. `@RequestParam(defaultValue = "") String q` reads `?q=`; missing means an empty string, never `null`.
2. `findByNameContainingIgnoreCase(..., Pageable)` returns a `Page`, so pagination still works. The SQL is
   `where upper(c1_0.name) like upper(?) escape '\'` with the value `%kala%`. Spring Data escapes `%` and `_`
   in the user's text.
3. The count query for the total uses the same `where`, so "Page 1 of N" is correct for the search.

Template — the form under `<h1>`, the new empty state, and `q` in the pagination links:

```html
<!-- src/main/resources/templates/clients/index.html — under <h1>Clients</h1> -->
<form method="get" th:action="@{/clients}" role="search">
  <input type="search" name="q" th:value="${q}" placeholder="Search by name"
         aria-label="Search clients by name">
  <button type="submit">Search</button>
</form>

<!-- replace the empty-state paragraph -->
<th:block th:if="${page.empty}">
  <p th:if="${q.isEmpty()}">No clients yet.</p>
  <p th:unless="${q.isEmpty()}" th:text="|No clients match &quot;${q}&quot;.|">No clients match "x".</p>
</th:block>
```

```html
<!-- in the pagination <nav>: add q to both links -->
<a th:if="${page.hasPrevious()}" th:href="@{/clients(page=${current - 1}, q=${q})}" rel="prev">← Previous</a>
<!-- ... -->
<a th:if="${page.hasNext()}" th:href="@{/clients(page=${current + 1}, q=${q})}" rel="next">Next →</a>
```

- `th:value="${q}"` keeps the typed text in the box, escaped as an attribute value.
- `@{/clients(page=..., q=${q})}` URL-encodes `q` correctly (spaces, `&`, `ü`).
- `method="get"`: searching changes nothing, so it is a GET; the URL can be bookmarked.

Check: `/clients?q=KALA` shows Kuressaare Kala AS; `/clients?q=<b>x</b>` shows the text
*No clients match "&lt;b&gt;x&lt;/b&gt;".* literally. With the 25 test clients, `/clients?q=test` has 3 pages
and the Next link keeps `q=test`.

</details>

<details>
<summary><strong>Advanced</strong> — Sortable columns with a whitelist</summary>

We stop letting `Pageable` read `sort` from the URL and build the `Pageable` ourselves from whitelisted
values.

```java
// src/main/java/ee/ta25/billable/client/ClientController.java — add the constant and replace index(...)
  /**
   * Public sort keys mapped to entity properties. Property names end up in the SQL ORDER BY and
   * cannot be bound parameters, so only these keys are accepted.
   */
  private static final Map<String, String> SORTABLE =
      Map.of("name", "name", "email", "email", "vat", "vatNumber");

  @GetMapping
  public String index(
      @RequestParam(defaultValue = "") String q,
      @RequestParam(defaultValue = "name") String sort,
      @RequestParam(defaultValue = "asc") String direction,
      @RequestParam(defaultValue = "1") int page,
      Model model) {
    String search = q.strip();
    String sortKey = SORTABLE.containsKey(sort) ? sort : "name";
    Sort.Direction dir = "desc".equals(direction) ? Sort.Direction.DESC : Sort.Direction.ASC;
    Pageable pageable =
        PageRequest.of(
            Math.max(page, 1) - 1, 10, Sort.by(dir, SORTABLE.get(sortKey)).and(Sort.by("id")));

    Page<Client> result =
        search.isEmpty()
            ? clients.findAll(pageable)
            : clients.findByNameContainingIgnoreCase(search, pageable);

    model.addAttribute("page", result);
    model.addAttribute("projectCounts", projectCountsFor(result.getContent()));
    model.addAttribute("q", search);
    model.addAttribute("sort", sortKey);
    model.addAttribute("direction", dir == Sort.Direction.DESC ? "desc" : "asc");
    return "clients/index";
  }
```

(Imports: `org.springframework.data.domain.PageRequest`, `org.springframework.data.domain.Sort`,
`org.springframework.web.bind.annotation.RequestParam`; remove the unused `PageableDefault` import.)

- Any `sort` value that is not a key of `SORTABLE` becomes `name`. Any `direction` except `desc` becomes
  ascending.
- `Math.max(page, 1) - 1` — we read `page` ourselves now, so `one-indexed-parameters` does not apply:
  the URL's 1-based number becomes the 0-based number `PageRequest.of` expects. `Math.max` turns `0` or
  a negative page into the first page; a negative number would make `PageRequest.of` throw.
- `Sort.by("id")` as a second key keeps the order stable when values are equal.

Template — replace the first three header cells:

```html
<!-- src/main/resources/templates/clients/index.html — the first three <th> in <thead> -->
<th scope="col">
  <a th:href="@{/clients(q=${q}, sort='name', direction=${sort == 'name' and direction == 'asc' ? 'desc' : 'asc'})}">
    Name <span th:if="${sort == 'name'}" th:text="${direction == 'asc' ? '▲' : '▼'}">▲</span>
  </a>
</th>
<th scope="col">
  <a th:href="@{/clients(q=${q}, sort='email', direction=${sort == 'email' and direction == 'asc' ? 'desc' : 'asc'})}">
    Email <span th:if="${sort == 'email'}" th:text="${direction == 'asc' ? '▲' : '▼'}">▲</span>
  </a>
</th>
<th scope="col">
  <a th:href="@{/clients(q=${q}, sort='vat', direction=${sort == 'vat' and direction == 'asc' ? 'desc' : 'asc'})}">
    VAT number <span th:if="${sort == 'vat'}" th:text="${direction == 'asc' ? '▲' : '▼'}">▲</span>
  </a>
</th>
```

Add `sort=${sort}, direction=${direction}` to both pagination links, and two hidden fields to the search
form:

```html
<input type="hidden" name="sort" th:value="${sort}">
<input type="hidden" name="direction" th:value="${direction}">
```

Why a whitelist (the text for your comment or `docs/`):

> Values in a query are sent as bound parameters (`where name = ?`), so user input never becomes SQL.
> Sort columns and directions cannot be parameters — they are part of the SQL text. Spring Data checks
> `Sort` properties against the entity, which blocks classic injection through `Pageable`, but an unknown
> property still causes a 500 error and lets users sort by any field you did not mean to expose. In a
> native query (`nativeQuery = true`) or with string-built JPQL, an unchecked sort value is a real SQL
> injection. A whitelist maps a few public names to real properties, and everything else falls back to a
> safe default.

Check: `/clients?sort=email&direction=desc` sorts by email Z–A; `/clients?sort=id;drop` and
`/clients?sort=createdAt` quietly sort by name; `/clients?page=-3` shows the first page.

</details>
