# 01 — Frameworks and the request lifecycle · Spring Boot

[← Concepts](README.md) · Starting point: nothing — this lesson creates the project · Estimated time in class: 2 h

---

## What we build today

- A new Spring Boot 4 project called `billable` (Java 25, Maven), generated at start.spring.io.
- A `compose.yaml` that runs PostgreSQL 17 in Docker — started automatically by Spring Boot's Docker
  Compose support.
- `application.properties` with the database connection and `ddl-auto=validate`; Flyway ready for the
  first migration in lesson 03.
- `TestcontainersConfiguration`: the test starts its own throw-away PostgreSQL 17 in Docker, so tests
  never need your development database.
- `templates/layout.html`: one Thymeleaf layout with Pico.css that every page uses.
- `common.HomeController` and `templates/home.html`: a home page that shows **Billable** and today's
  date, where the date is supplied by the controller.
- A Git repository on GitHub with the first commits.

When you are done, `http://localhost:8080` shows a clean, styled page with a navigation bar, the title
"Billable" and a sentence like "Today is 30.09.2026."

---

## Step 0 — Check your tools

**Why:** most first-lesson problems are a missing or wrong JDK. Checking first takes two minutes.

```bash
java -version
docker --version
docker compose version
git --version
```

Also check which JDK your terminal uses:

```bash
# macOS / Linux
echo $JAVA_HOME

# Windows PowerShell
echo $env:JAVA_HOME
```

**Check it works:**

| Command | You need |
|---|---|
| `java -version` | `openjdk version "25…"` (for example Eclipse Temurin 25) |
| `JAVA_HOME` | a path to the **same** JDK 25 folder (not a JRE, not an old JDK) |
| `docker compose version` | `Docker Compose version v2.x` (note: `docker compose`, with a space) |
| `git --version` | any recent version |

You do **not** need to install Maven. The project contains the **Maven wrapper** (`mvnw`), which
downloads the right Maven version the first time you use it.

Start **Docker Desktop** and wait until it says it is running. `docker ps` must print a table header,
not an error.

---

## Step 1 — Generate the project

**Why:** Spring Initializr creates a project with a correct `pom.xml` for the current Spring Boot
version. Writing a `pom.xml` by hand, or copying one from a tutorial, is the fastest way to get an
outdated or broken set-up.

Open **https://start.spring.io** and fill in the form:

| Field | Value |
|---|---|
| Project | **Maven** |
| Language | **Java** |
| Spring Boot | the newest **4.x** version that is not `SNAPSHOT` or `M…`/`RC…` |
| Group | `ee.ta25` |
| Artifact | `billable` |
| Name | `billable` |
| Description | `Time tracking and invoicing` |
| Package name | `ee.ta25.billable` |
| Packaging | **Jar** |
| Configuration | **Properties** |
| Java | **25** |

Click **ADD DEPENDENCIES** and add these nine:

| Dependency | What it gives us | Used from lesson |
|---|---|---|
| **Spring Web** | Spring MVC + embedded Tomcat | 01 |
| **Thymeleaf** | HTML templates | 01 |
| **Validation** | Jakarta Bean Validation (`@NotBlank`, `@Email`…) | 05 |
| **Spring Data JPA** | Hibernate + repositories | 03 |
| **Flyway Migration** | Versioned SQL migrations | 03 |
| **PostgreSQL Driver** | JDBC driver for PostgreSQL | 01 |
| **Docker Compose Support** | Starts `compose.yaml` when the app starts | 01 |
| **Spring Boot DevTools** | Automatic restart when code changes | 01 |
| **Testcontainers** | Tests start their own PostgreSQL in Docker | 01 |

Click **GENERATE**. Unzip `billable.zip` into your projects folder and open the `billable` folder in
IntelliJ IDEA (File → Open → select the folder that contains `pom.xml`). IntelliJ imports the Maven
project; the first import downloads dependencies and takes a few minutes.

**What happens:**

1. Initializr chooses dependency versions that are tested together (the "Spring Boot BOM" in the
   parent POM). That is why the dependencies in `pom.xml` have no `<version>` tags.
2. It generates `BillableApplication.java` (the entry point), `application.properties`, one test,
   the Maven wrapper, a `.gitignore`, and — because you picked Docker Compose Support and
   PostgreSQL — a `compose.yaml`.
3. Because you picked Testcontainers and PostgreSQL, it also adds the Testcontainers PostgreSQL module
   and generates two test classes: `TestcontainersConfiguration` and `TestBillableApplication`. We
   look at them in Step 9.

---

## Step 2 — Check `pom.xml` and take a tour

**Why:** Spring Boot 4 renamed and split several modules. If a tutorial or an AI tool gave you a
`pom.xml`, it is probably written for Boot 3. Check that yours has the Boot 4 names.

Open `pom.xml` and find the `<dependencies>` section. You should see (the order may differ):

| Look for | Notes |
|---|---|
| `<parent>` … `spring-boot-starter-parent` with version `4.x.x` | the Boot version |
| `<java.version>25</java.version>` | in `<properties>` |
| `spring-boot-starter-webmvc` | Boot 4 name. In Boot 3 it was `spring-boot-starter-web`. |
| `spring-boot-starter-thymeleaf` | |
| `spring-boot-starter-validation` | |
| `spring-boot-starter-data-jpa` | |
| a **Flyway** starter or module (artifact id contains `flyway`) **and** `flyway-database-postgresql` | Both are needed. Without the Spring Boot Flyway module, migrations silently never run. Without `flyway-database-postgresql`, Flyway says it does not support PostgreSQL. |
| `postgresql` (scope `runtime`) | the JDBC driver |
| `spring-boot-docker-compose` | |
| `spring-boot-devtools` | |
| `spring-boot-testcontainers`, `testcontainers-junit-jupiter`, `testcontainers-postgresql` (scope `test`) | Testcontainers **2.x** names. Boot 3 tutorials show `junit-jupiter` and `postgresql` from `org.testcontainers`; with Boot 4 Maven then says the version is missing. |
| other test dependencies (scope `test`) | Initializr adds a matching test starter for each feature, for example `spring-boot-starter-webmvc-test` |

If something is missing, the easiest fix is to generate the project again rather than to edit the
POM by hand.

Now the folder tour:

```
billable/
├── pom.xml                         ← dependencies and build configuration
├── mvnw, mvnw.cmd, .mvn/           ← Maven wrapper (use ./mvnw, never a global mvn)
├── compose.yaml                    ← Docker services (we replace it in Step 3)
├── src/main/java/ee/ta25/billable/
│   └── BillableApplication.java    ← entry point: main() starts Spring and Tomcat
├── src/main/resources/
│   ├── application.properties      ← configuration
│   ├── templates/                  ← Thymeleaf views (.html)
│   └── static/                     ← static files served as-is (CSS, images)
├── src/test/java/ee/ta25/billable/
│   ├── BillableApplicationTests.java  ← one test: "the application context starts"
│   ├── TestcontainersConfiguration.java  ← a PostgreSQL container for tests (Step 9)
│   └── TestBillableApplication.java   ← we delete it in Step 9
└── target/                         ← build output (NOT committed)
```

Open `BillableApplication.java`. (Initializr indents its files with tabs. The listings in this guide
use two spaces — the Google Java Style we adopt in lesson 02, where a formatter converts the generated
files for you.)

```java
// src/main/java/ee/ta25/billable/BillableApplication.java (generated)
package ee.ta25.billable;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class BillableApplication {

  public static void main(String[] args) {
    SpringApplication.run(BillableApplication.class, args);
  }
}
```

**What happens** when you run it:

1. `@SpringBootApplication` switches on three things: it marks this class as configuration, turns on
   **auto-configuration** (Spring looks at the jars on the classpath and configures what it finds —
   Thymeleaf jar present → Thymeleaf view resolver created), and turns on **component scanning** for
   this package and all sub-packages.
2. Component scanning is why every class must live **under** `ee.ta25.billable`. A controller in
   package `ee.ta25.other` would never be found.
3. `SpringApplication.run(...)` creates the **application context** — a container that holds all the
   objects (beans) Spring manages — and starts the embedded Tomcat web server.

This `main` method is the only `main` you will ever write. After that, Spring calls your code
(inversion of control).

---

## Step 3 — PostgreSQL in Docker

**Why:** everybody gets exactly the same database version without installing PostgreSQL, and you can
reset it in seconds.

Initializr generated a `compose.yaml` with example names (`mydatabase`, `myuser`) and the `latest`
image tag. **Replace the whole file** with the agreed version, which is identical in both tracks:

```yaml
# compose.yaml
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_DB: billable
      POSTGRES_USER: billable
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U billable -d billable"]
      interval: 5s
      timeout: 5s
      retries: 10

volumes:
  pgdata:
```

Start it once by hand to see that it works:

```bash
docker compose up -d
docker compose ps
```

**What happens:**

1. `image: postgres:17` — the official PostgreSQL image, major version 17. We pin the version: with
   `latest` you would silently get version 18 one day, which stores data in a different folder and does
   not start with this file.
2. `environment` — on the **first** start the image creates database `billable`, user `billable`,
   password `secret`. Later starts ignore these values, because the data already exists.
3. `ports: "5432:5432"` — the generated file had only `'5432'`, which means "a random free port on the
   laptop". We use a fixed port so that `psql` and database tools can find it at `localhost:5432`.
4. `volumes` — the data lives in the Docker volume `pgdata` and survives restarts. `docker compose down -v`
   deletes it.
5. `healthcheck` — Docker runs `pg_isready` regularly, so `docker compose ps` shows `(healthy)` when the
   database accepts connections.

**Check it works:**

```bash
docker compose exec postgres psql -U billable -d billable -c "select version();"
```

You see a line starting with `PostgreSQL 17.`

**Docker Compose Support — what it does.** From now on you do not have to run `docker compose up`
yourself. When the application starts, the `spring-boot-docker-compose` module:

1. finds `compose.yaml` in the working directory,
2. runs `docker compose up` and waits until the services are ready,
3. recognises the `postgres` image, reads `POSTGRES_USER`, `POSTGRES_PASSWORD` and `POSTGRES_DB`, and
   configures the database connection **automatically**,
4. runs `docker compose stop` when the application shuts down (the data stays in the volume).

It does this only when you *run* the application. It is **skipped in tests** by default: tests get
their own database from Testcontainers (Step 9).

---

## Step 4 — Configure the application

**Why:** Docker Compose Support connects the running app for us, and in tests Testcontainers does the
same. Writing the connection down anyway documents the values, and the app still starts if you run
PostgreSQL some other way (for example on a server, with `DB_PASSWORD` set).

Replace `src/main/resources/application.properties`:

```properties
# src/main/resources/application.properties
spring.application.name=billable

# Database - same values as compose.yaml.
# When the app runs, Docker Compose Support supplies these automatically;
# in tests, Testcontainers does. Without either, Spring reads them from here.
spring.datasource.url=jdbc:postgresql://localhost:5432/billable
spring.datasource.username=billable
spring.datasource.password=${DB_PASSWORD:secret}

# Flyway owns the schema. Hibernate only checks that entities match it.
spring.jpa.hibernate.ddl-auto=validate
```

Then create the folder for migrations, with an empty placeholder file so Git keeps the folder:

```bash
mkdir -p src/main/resources/db/migration
touch src/main/resources/db/migration/.gitkeep
```

(Windows PowerShell: `New-Item -ItemType Directory -Force src/main/resources/db/migration` and
`New-Item src/main/resources/db/migration/.gitkeep`.)

**What happens:**

1. `spring.datasource.*` — the JDBC URL has the form `jdbc:postgresql://host:port/database`.
2. `${DB_PASSWORD:secret}` is a **placeholder with a default**: Spring uses the environment variable
   `DB_PASSWORD` if it exists, otherwise `secret`. On a real server you would set `DB_PASSWORD` and the
   real password never appears in Git. (Spring also lets any property be overridden directly, for
   example with the environment variable `SPRING_DATASOURCE_PASSWORD`.)
3. `ddl-auto=validate` — Hibernate can create tables from your entity classes (`create`, `update`).
   We **forbid** that. Tables are created only by Flyway migration files, which are versioned, reviewed
   and repeatable. `validate` makes Hibernate check at start-up that the entities match the tables and
   refuse to start if they do not. Today there are no entities, so there is nothing to check.
4. **Flyway with no migrations.** Flyway runs at start-up, looks in `db/migration`, finds no `.sql`
   files and does nothing. We made this choice deliberately instead of adding an empty `V1__init.sql`:
   an empty migration would stay in the history for ever and mean nothing. The first real migration,
   `V1__…`, arrives in lesson 03 when we create the `clients` table. Flyway ignores the `.gitkeep` file
   because it only reads files ending in `.sql`.

**Timezone.** Billable works in Estonian time. We do not change the server's timezone; instead, code
that needs "today" asks for it in `Europe/Tallinn` explicitly (next step). In lesson 10 this becomes a
`Clock` bean.

---

## Step 5 — First start

```bash
./mvnw spring-boot:run
```

On Windows PowerShell: `.\mvnw spring-boot:run`. In IntelliJ you can instead press the green ▶ next to
`main` in `BillableApplication`.

The first run downloads Maven and all dependencies — it can take several minutes.

**Check it works:** in the log, look for lines similar to these (exact wording depends on versions):

```
... DockerComposeLifecycleManager : Using Docker Compose file .../billable/compose.yaml
... o.f.core.internal.command.DbMigrate : Schema "public" is up to date. No migration necessary.
... TomcatWebServer : Tomcat started on port 8080 (http) with context path '/'
... BillableApplication : Started BillableApplication in 4.1 seconds
```

Flyway may also print a warning like `No migrations found. Are your locations set up correctly?` That
is expected today.

Open `http://localhost:8080`. You see a **Whitelabel Error Page** with `status=404`. That is correct
and useful: the whole lifecycle worked — Tomcat received the request, `DispatcherServlet` asked every
`HandlerMapping` for a handler for `GET /`, found none, and Spring's error handling produced a 404
page. We fix that now.

Leave the application running. With DevTools, it restarts itself when compiled classes change.

---

## Step 6 — The layout

**Why:** every page shares the same `<head>`, navigation and styling. Writing it once in a layout
means a menu change is one edit, not twenty.

Thymeleaf shares HTML with **fragments**: a named piece of a template that other templates can use.
A fragment can have **parameters**, and a parameter can itself be a piece of HTML. We use that for one
layout that wraps the whole page: each page hands over its `<title>` and its `<main>`, and the layout
puts them in the right places. Create `src/main/resources/templates/layout.html`:

```html
<!-- src/main/resources/templates/layout.html -->
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org" th:fragment="layout(title, content)">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title th:replace="${title}">Billable</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@picocss/pico@2/css/pico.min.css">
</head>
<body>

<header class="container">
  <nav>
    <ul>
      <li><strong>Billable</strong></li>
    </ul>
    <ul>
      <li><a th:href="@{/}">Home</a></li>
    </ul>
  </nav>
</header>

<div class="container" th:if="${status}">
  <article role="status" th:text="${status}">Saved.</article>
</div>

<main class="container" th:replace="${content}">
  <p>The page content goes here.</p>
</main>

<footer class="container">
  <small>Billable · TA-25</small>
</footer>

</body>
</html>
```

**What happens:**

1. `xmlns:th="http://www.thymeleaf.org"` declares the `th:` prefix, so IntelliJ understands the
   attributes. Thymeleaf itself works without it, but keep it for editor support.
2. `th:fragment="layout(title, content)"` on `<html>` makes the **whole file** a fragment called
   `layout` with two parameters. Both will be pieces of HTML from the page: its `<title>` element and
   its `<main>` element.
3. `th:replace="${title}"` replaces the layout's placeholder `<title>` with the page's `<title>`.
   `th:replace="${content}"` does the same for `<main>`. Everything else — head, navigation, flash
   message, footer — is written only here.
4. The `<link>` loads Pico.css from jsDelivr. Pico styles plain HTML (`nav`, `article`, `table`,
   `form`) without CSS classes; `class="container"` centres the content.
5. `th:href="@{/}"` is a **link expression**. It adds the application's context path if there is one,
   so links keep working if the app is ever deployed under `/billable/`.
6. The flash box shows a one-time message (for example "Client saved.") only when a model attribute
   `status` exists (`th:if`). It is in the layout, so every page shows it without doing anything.
   Nothing sends such messages yet — lesson 05 does — but the layout is ready.
7. `th:text` replaces the element's text and always escapes HTML, so user input cannot inject
   `<script>` tags.
8. The text inside elements (`Billable`, `Saved.`) is only a **prototype value**: you see it if you
   open the file directly in a browser; Thymeleaf replaces it when rendering. This is called *natural
   templating*.

---

## Step 7 — HomeController and the home view

**Why:** our first complete MVC request. The **controller** decides what data the page needs; the
**view** only displays it.

Create the package `common` (right-click `ee.ta25.billable` → New → Package → `common`) and the
controller in it:

```java
// src/main/java/ee/ta25/billable/common/HomeController.java
package ee.ta25.billable.common;

import java.time.LocalDate;
import java.time.ZoneId;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class HomeController {

  private static final ZoneId TALLINN = ZoneId.of("Europe/Tallinn");

  @GetMapping("/")
  public String home(Model model) {
    model.addAttribute("today", LocalDate.now(TALLINN));
    return "home";
  }
}
```

Create `src/main/resources/templates/home.html`:

```html
<!-- src/main/resources/templates/home.html -->
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      th:replace="~{layout :: layout(~{::title}, ~{::main})}">
<head>
  <title>Home · Billable</title>
</head>
<body>
<main class="container">
  <hgroup>
    <h1>Billable</h1>
    <p>Time tracking and invoicing for freelancers.</p>
  </hgroup>

  <p>Today is <span th:text="${#temporals.format(today, 'dd.MM.yyyy')}">01.01.2026</span>.</p>
</main>
</body>
</html>
```

**What happens:**

1. `@Controller` marks the class as a web controller. Component scanning finds it at start-up (it is
   under `ee.ta25.billable`) and Spring creates one instance of it — a **bean**.
2. `@GetMapping("/")` registers the method with `RequestMappingHandlerMapping`: "for `GET /`, call
   `home`". This annotation *is* the route; Spring has no separate routes file.
3. `Model model` — you never create the `Model` yourself. Spring sees the parameter type and passes
   one in (inversion of control again). Whatever you put into it is available in the template.
4. `LocalDate.now(TALLINN)` asks for today's date **in Tallinn**, not in the server's timezone. A
   server in a data centre often runs on UTC; between 00:00 and 03:00 Tallinn time, UTC is still
   "yesterday".
5. `return "home"` returns a **view name**, not HTML. The `ThymeleafViewResolver` turns it into
   `classpath:/templates/home.html`.
6. In the template, `th:replace="~{layout :: layout(~{::title}, ~{::main})}"` on `<html>` replaces the
   whole page with the `layout` fragment from `layout.html`. `~{::title}` and `~{::main}` mean "the
   `<title>` element and the `<main>` element of **this** file"; they become the layout's `title` and
   `content` parameters. So a page contains only what is special about it: its title and its main
   content. The `<head>` and `<body>` around them are there so that the file is still valid HTML.
7. `${#temporals.format(today, 'dd.MM.yyyy')}` formats the `LocalDate` the Estonian way (`30.09.2026`).
   `#temporals` is a Thymeleaf utility object for `java.time` types. Formatting for display is the
   view's job; *choosing* the date is the controller's job.

Why does the controller supply the date instead of the template? Because the view should not make
decisions, and because in lesson 12 we will replace "now" with a fixed time in tests. That is only easy
if "now" enters the page in one known place.

**Note on the class layout:** `HomeController` goes in `common` because it does not belong to any
business feature. Later features get their own packages (`client`, `project`, …) — see the code map in
PROJECT.md.

---

## Step 8 — Run it and watch the request

If the application is still running from Step 5, DevTools restarts it when the classes are recompiled:

- **IntelliJ:** Build → Build Project (Ctrl+F9 / ⌘F9). You can switch on *Settings → Build, Execution,
  Deployment → Compiler → Build project automatically* to make this happen on save.
- **Terminal:** run `./mvnw compile` in a second terminal.

Or simply stop the application (Ctrl+C) and start it again with `./mvnw spring-boot:run`.

**Check it works:** open `http://localhost:8080`. You see a styled page with a nav bar, the heading
**Billable**, and "Today is …" with today's date in `dd.mm.yyyy` format.

Now look at the request:

1. Open the developer tools (F12) → **Network** → reload. Click the request for `localhost`. You see
   Request Method `GET`, Status `200`, response header `Content-Type: text/html;charset=UTF-8`.
2. Visit `http://localhost:8080/nothing-here` → Whitelabel Error Page, status 404. No handler matched.

**See the lifecycle in the log.** Stop the app and start it with web DEBUG logging:

```bash
./mvnw spring-boot:run -Dspring-boot.run.arguments=--logging.level.org.springframework.web=DEBUG
```

Reload `http://localhost:8080`. The log shows lines like:

```
o.s.web.servlet.DispatcherServlet        : GET "/", parameters={}
s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped to ee.ta25.billable.common.HomeController#home(Model)
o.s.web.servlet.DispatcherServlet        : Completed 200 OK
```

This is the request lifecycle written out by the framework itself: the front controller received it,
the handler mapping chose your method, and the response was completed with 200. Keep this command for
the independent Task 2. Restart normally afterwards; DEBUG logging is very noisy.

---

## Step 9 — A test database with Testcontainers

**Why:** a test must not depend on your development database being started, and must not change its
data. **Testcontainers** starts a fresh PostgreSQL in Docker for the test run and removes it
afterwards. The same test then also works on the CI server in lesson 02.

Initializr generated `TestcontainersConfiguration`. Change it to match this listing: the class becomes
`public` (tests in other packages import it from lesson 14 on), the image is pinned to the same major
version as `compose.yaml`, and the bean gets a shorter name.

```java
// src/test/java/ee/ta25/billable/TestcontainersConfiguration.java
package ee.ta25.billable;

import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.context.annotation.Bean;
import org.testcontainers.postgresql.PostgreSQLContainer;

/** One PostgreSQL 17 container for the tests; Spring points the DataSource at it. */
@TestConfiguration(proxyBeanMethods = false)
public class TestcontainersConfiguration {

  @Bean
  @ServiceConnection
  PostgreSQLContainer postgres() {
    return new PostgreSQLContainer("postgres:17-alpine");
  }
}
```

Initializr also generated the test that uses it:

```java
// src/test/java/ee/ta25/billable/BillableApplicationTests.java
package ee.ta25.billable;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;

@SpringBootTest
@Import(TestcontainersConfiguration.class)
class BillableApplicationTests {

  @Test
  void contextLoads() {}
}
```

Delete `src/test/java/ee/ta25/billable/TestBillableApplication.java`. It starts the app with a
container database instead of `compose.yaml`; we always use `compose.yaml` for running the app, so one
way is enough.

Docker Desktop must be running. Run the tests:

```bash
./mvnw test
```

**What happens:**

1. `@SpringBootTest` starts the **whole application context** — including the database connection and
   Flyway — but without a web server on port 8080.
2. `@Import(TestcontainersConfiguration.class)` adds the configuration class to the test context.
   `@TestConfiguration` means "configuration that only tests use"; the running app never sees it.
3. The `PostgreSQLContainer` bean starts a container from the image `postgres:17-alpine` (a small
   PostgreSQL 17 image; the first run downloads it). Its port is random, so it never collides with
   your development database on 5432.
4. `@ServiceConnection` reads the container's address, user and password and gives them to Spring's
   `DataSource`. They win over `spring.datasource.*` in `application.properties`. Flyway runs on the
   empty container database.
5. The test method is empty. If the context starts without an exception, the test passes. It is a
   "smoke test": it catches broken configuration.
6. When the tests end, Testcontainers removes the container. Docker Compose Support is skipped in
   tests, so your development database is not touched — it does not even need to be running.

> **Version pitfall.** Spring Boot 4 uses **Testcontainers 2.x**. The class is
> `org.testcontainers.postgresql.PostgreSQLContainer` and takes no `<?>`. Boot 3 tutorials import
> `org.testcontainers.containers.PostgreSQLContainer`; that old class still exists but is deprecated,
> and `@ServiceConnection` does not recognise it.

**Check it works:** stop your development database (`docker compose stop`) and run `./mvnw test`. The
log shows lines like `Creating container for image: postgres:17-alpine` and `Container
postgres:17-alpine started`, and the output ends with `Tests run: 1, Failures: 0, Errors: 0` and
`BUILD SUCCESS`. The application itself still uses `compose.yaml`: the next `./mvnw spring-boot:run`
starts it again.

---

## Step 10 — Git and GitHub

**Why:** your Git history is evidence for ÕV7 and ÕV9. It starts today.

Initializr already created a `.gitignore` (it excludes `target/`, IDE folders and `HELP.md`). Open it
and add one line at the end, so that a local `.env` file (some tools create one) is never committed:

```gitignore
# .gitignore — add at the end
.env
```

Create the repository:

```bash
git init -b main
git status --short
```

**Check it works:** the list contains `pom.xml`, `mvnw`, `compose.yaml`, `src/` and `.gitignore`. It
must **not** contain `target/` or `.idea/`.

**Windows users:** make sure `mvnw` stays executable for Linux (the CI server in lesson 02 runs it):

```bash
git add mvnw
git update-index --chmod=+x mvnw
```

Commit:

```bash
git add .
git commit -m "chore: create Spring Boot project with PostgreSQL and home page"
```

The message format `type(scope): summary` is **Conventional Commits** — lesson 02 explains it. From
the next lesson on we commit after each step.

**Push to GitHub:**

1. On github.com click **New repository**. Name: `billable`. Do **not** tick "Add a README",
   ".gitignore" or "license" — the repository must be empty, otherwise the first push is rejected.
2. Run the commands GitHub shows under "…or push an existing repository from the command line":

```bash
git remote add origin git@github.com:<your-user>/billable.git
git branch -M main
git push -u origin main
```

Without SSH keys, use the HTTPS URL (`https://github.com/<your-user>/billable.git`); Git will open a
browser login.

**Check it works:** the repository page on GitHub shows your files and no `target/` folder. Share the
repository with the teacher (Settings → Collaborators) if it is private.

---

## Troubleshooting

| Error / symptom | Cause | Fix |
|---|---|---|
| `release version 25 not supported` / `invalid target release: 25` | Maven runs with an older JDK | `JAVA_HOME` points to an old JDK. Set it to the JDK 25 folder and open a new terminal. In IntelliJ: File → Project Structure → SDK = 25. |
| `JAVA_HOME is not defined correctly` / `JAVA_HOME environment variable is not defined` | The wrapper cannot find a JDK | Set `JAVA_HOME` to the JDK folder (the one that contains `bin/java`), not to `bin` itself. |
| `./mvnw: Permission denied` | The file lost its executable bit (common after unzipping on some systems) | `chmod +x mvnw`. For Git: `git update-index --chmod=+x mvnw`. |
| `Connection to localhost:5432 refused` in a test | The test does not use the container: `@Import(TestcontainersConfiguration.class)` is missing, so Spring falls back to `spring.datasource.url` | Add the import (Step 9). |
| `Could not find a valid Docker environment` in a test | Docker Desktop is not running (Testcontainers needs it) | Start Docker Desktop; `docker ps` must work in a terminal. |
| `'dependencies.dependency.version' for org.testcontainers:postgresql:jar is missing` | Boot 3 artifact names copied from a tutorial | Use `testcontainers-postgresql` and `testcontainers-junit-jupiter` (Step 2). |
| `Cannot connect to the Docker daemon` / `Docker is not running` at start-up | Docker Desktop is not running | Start Docker Desktop and wait until it is ready. |
| `Bind for 0.0.0.0:5432 failed: port is already allocated` | A local PostgreSQL or another container uses 5432 | Stop it (`docker ps`, stop the other container; stop the local PostgreSQL service). Or map `"5433:5432"` in `compose.yaml` and change the URL to `localhost:5433`. |
| `password authentication failed for user "billable"` | The volume was created earlier with other credentials (e.g. Initializr's `myuser`) | `docker compose down -v`, then start again — the database is re-created with our values. |
| `Port 8080 was already in use` | Another app (or a second copy of Billable) is running | Stop the other one, or add `server.port=8081` to `application.properties` temporarily. |
| `Unsupported Database: PostgreSQL 17…` from Flyway | `flyway-database-postgresql` missing from `pom.xml` | Regenerate at start.spring.io, or add the dependency (no version — the parent manages it). |
| Migrations never run, no Flyway lines in the log | Spring Boot Flyway module missing (Boot 3 tutorials only add `flyway-core`) | Check `pom.xml` as in Step 2. |
| `Error resolving template [home]` | File name or location wrong | Must be `src/main/resources/templates/home.html`; the controller returns `"home"` without `.html`. |
| Whitelabel 404 on `/` although the controller exists | The controller is outside `ee.ta25.billable`, or annotated `@RestController`/missing `@Controller` | Check the package line and the annotation. |
| A page shows the nav and footer, but its own content (or its tab title) is missing | The page has no `<main>` (or `<title>`) element, so `~{::main}` (`~{::title}`) selects nothing | Put the content in `<main class="container">` and the title in `<head><title>…</title></head>` (Step 7). |
| Page shows the raw text `01.01.2026` | You opened the HTML file directly in the browser | Open `http://localhost:8080`, not the file. |
| Template changes do not show | Resources not recompiled | Build Project in IntelliJ, or `./mvnw compile`. |

---

## Recap

- `pom.xml` — generated by Initializr; checked for Boot 4 module names (`spring-boot-starter-webmvc`,
  Flyway module + `flyway-database-postgresql`).
- `compose.yaml` — PostgreSQL 17, same settings in both tracks; started automatically by Docker
  Compose Support when the app runs.
- `application.properties` — explicit datasource (used when neither Compose Support nor
  Testcontainers supplies one), password placeholder, `ddl-auto=validate`; `db/migration/` is ready
  but empty until lesson 03.
- `TestcontainersConfiguration` — a `@ServiceConnection` `PostgreSQLContainer` (`postgres:17-alpine`);
  `contextLoads` imports it, so tests run on their own throw-away database.
- `templates/layout.html` — one `layout(title, content)` fragment: head, nav, flash message and footer,
  written once.
- `common/HomeController.java` — the C: chooses the data (today in Tallinn) and the view name.
- `templates/home.html` — the V: displays it; `th:replace="~{layout :: layout(~{::title}, ~{::main})}"`
  hands its `<title>` and `<main>` to the layout.
- A request travels: Tomcat → filters → `DispatcherServlet` → `HandlerMapping` → `HomeController` →
  `ThymeleafViewResolver` → `home.html` → response.

---

## Independent work — solutions

Try each task yourself first.

<details>
<summary>Task 1 — About page (Basic)</summary>

Add a second method to `HomeController`. Both pages are simple information pages, so they belong
together.

```java
// src/main/java/ee/ta25/billable/common/HomeController.java
package ee.ta25.billable.common;

import java.time.LocalDate;
import java.time.ZoneId;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class HomeController {

  private static final ZoneId TALLINN = ZoneId.of("Europe/Tallinn");

  @GetMapping("/")
  public String home(Model model) {
    model.addAttribute("today", LocalDate.now(TALLINN));
    return "home";
  }

  @GetMapping("/about")
  public String about() {
    return "about";
  }
}
```

A handler method does not need a `Model` parameter if it passes no data.

```html
<!-- src/main/resources/templates/about.html -->
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      th:replace="~{layout :: layout(~{::title}, ~{::main})}">
<head>
  <title>About · Billable</title>
</head>
<body>
<main class="container">
  <h1>About Billable</h1>
  <p>
    Billable is a time-tracking and invoicing tool for freelancers and small agencies.
    You log the hours you work for your clients, and Billable turns them into correct
    invoices with VAT.
  </p>
  <p>It is built step by step in the M8 module at Kuressaare Ametikool.</p>
</main>
</body>
</html>
```

Check: `http://localhost:8080/about` shows the page; the tab title is "About · Billable".

```bash
git add .
git commit -m "feat(home): add about page"
```

</details>

<details>
<summary>Task 2 — Trace a request (Basic)</summary>

Write it in your own words — this shows the expected level of detail.

```markdown
<!-- docs/request-lifecycle.md -->
# Request lifecycle: GET /about

Before any request: `BillableApplication.main()` calls `SpringApplication.run()`. Spring scans
`ee.ta25.billable`, creates a bean for `HomeController`, registers its `@GetMapping` methods and starts
embedded Tomcat on port 8080. This happens once.

1. **Tomcat** receives `GET /about` on port 8080.
2. **Servlet filters** run (for example `CharacterEncodingFilter`), then the request reaches
   **`DispatcherServlet`**, Spring MVC's front controller.
3. **`RequestMappingHandlerMapping`** finds the handler: `HomeController#about()`.
4. **`HomeController.about()`** (`src/main/java/ee/ta25/billable/common/HomeController.java`) returns
   the view name `"about"`.
5. **`ThymeleafViewResolver`** resolves `"about"` to `src/main/resources/templates/about.html`.
6. **Thymeleaf** renders `about.html`; `th:replace` on `<html>` wraps its `<title>` and `<main>` in the
   `layout` fragment from `templates/layout.html` (head, nav, footer).
7. The HTML is written to the response with status 200 and goes back through the filters and Tomcat
   to the browser.

Evidence (DEBUG log):

    o.s.web.servlet.DispatcherServlet        : GET "/about", parameters={}
    s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped to ee.ta25.billable.common.HomeController#about()
    o.s.web.servlet.DispatcherServlet        : Completed 200 OK

MVC in this request: `HomeController` is the controller and `about.html` is the view. There is no
model data because the page only shows fixed text.
```

```bash
git add docs/request-lifecycle.md
git commit -m "docs: describe request lifecycle"
```

</details>

<details>
<summary>Task 3 — Links from mappings and an active nav link (Intermediate)</summary>

**How Spring differs from Laravel's named routes.** Spring MVC routes have no names. The URL is
written in the `@GetMapping` annotation, and templates link with `@{/about}`. Link expressions add the
context path for you but do not protect you from a changed mapping: if you change `/about` to
`/about-billable`, you must change the link too. The closest thing to named routes is Thymeleaf's
`#mvc.url` helper, which builds a URL from the controller method:
`th:href="${#mvc.url('HC#about').build()}"` (`HC` = the capital letters of `HomeController`). It is
rarely used in practice, because the short form is easier to read; mention the trade-off in your
`docs/request-lifecycle.md`.

**Active link.** The template must know the current path. **Version pitfall:** older tutorials use
`${#httpServletRequest.requestURI}` or `${#request...}` in the template. Those objects were **removed
in Thymeleaf 3.1** (used by Spring Boot 3 and 4). Instead, put the path into every model with a
`@ControllerAdvice`:

```java
// src/main/java/ee/ta25/billable/common/CurrentPathAdvice.java
package ee.ta25.billable.common;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ModelAttribute;

/** Adds the current request path to every model, so the layout can highlight the active link. */
@ControllerAdvice
public class CurrentPathAdvice {

  @ModelAttribute("currentPath")
  public String currentPath(HttpServletRequest request) {
    return request.getRequestURI();
  }
}
```

Replace the `<header>` in `templates/layout.html`:

```html
<!-- src/main/resources/templates/layout.html — the <header> -->
<header class="container">
  <nav>
    <ul>
      <li><strong>Billable</strong></li>
    </ul>
    <ul>
      <li>
        <a th:href="@{/}" th:aria-current="${currentPath == '/'} ? 'page' : null">Home</a>
      </li>
      <li>
        <a th:href="@{/about}" th:aria-current="${currentPath == '/about'} ? 'page' : null">About</a>
      </li>
    </ul>
  </nav>
</header>
```

**What happens:**

1. A `@ControllerAdvice` class applies to all controllers. Its `@ModelAttribute` method runs before
   every handler method and adds `currentPath` to the model.
2. `HttpServletRequest` is injected by Spring as a method parameter — note `jakarta.servlet`, not
   `javax.servlet` (the old package name from before Spring Boot 3).
3. `th:aria-current="… ? 'page' : null"` sets the attribute to `page` on the current link. Thymeleaf
   turns any `th:xyz` into a plain `xyz` attribute. When the value is `null`, Thymeleaf does not write
   the attribute at all.
4. Pico.css styles `aria-current="page"`, and screen readers announce it.

```bash
git add .
git commit -m "feat(home): add about link with active state to navigation"
```

</details>

<details>
<summary>Task 4 — Request timing filter (Advanced)</summary>

```java
// src/main/java/ee/ta25/billable/common/RequestTimingFilter.java
package ee.ta25.billable.common;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.time.Duration;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

/** Logs the method, path, status and duration of every request. */
@Component
public class RequestTimingFilter extends OncePerRequestFilter {

  private static final Logger log = LoggerFactory.getLogger(RequestTimingFilter.class);

  @Override
  protected void doFilterInternal(
      HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
      throws ServletException, IOException {
    long start = System.nanoTime();
    try {
      filterChain.doFilter(request, response);
    } finally {
      long millis = Duration.ofNanos(System.nanoTime() - start).toMillis();
      log.info(
          "{} {} -> {} in {} ms",
          request.getMethod(),
          request.getRequestURI(),
          response.getStatus(),
          millis);
    }
  }
}
```

**What happens:**

1. `@Component` makes the class a bean. Spring Boot automatically registers every bean of type
   `Filter` in Tomcat's filter chain — no extra configuration.
2. `OncePerRequestFilter` guarantees the filter runs once per request, even when the request is
   forwarded internally (for example to the `/error` page).
3. Code **before** `filterChain.doFilter(...)` runs on the way in; code **after** it runs on the way out,
   after `DispatcherServlet`, your controller and Thymeleaf have finished. The filter wraps the rest
   of the lifecycle, just like Laravel middleware.
4. `finally` makes sure the line is logged even if the controller throws an exception.
5. `System.nanoTime()` is the right clock for measuring durations (it never jumps when the system
   time changes). `Duration.ofNanos(...).toMillis()` converts the difference to milliseconds without a
   hand-written `/ 1_000_000`. `{}` placeholders in SLF4J are filled in only if the log level is
   enabled.

**Why not a header?** By the time `doFilter` returns, Thymeleaf has usually already written the HTML
and the response is **committed** — the status line and headers have been sent to the browser. Setting
a header after that is silently ignored. To add a header you would have to set it *before* the body is
written (for example in a `HandlerInterceptor.postHandle`, which runs before the view is rendered, so
it would not measure rendering time) or buffer the whole response. Logging avoids the problem.

**In a real project** you would not write this filter yourself. Spring Boot Actuator records the
duration of every request as the metric `http.server.requests` (by method, URI and status). We write
the filter here to see where a filter sits in the lifecycle.

**Check it works:** reload a few pages. The application log shows lines like:

```
INFO ... e.t.billable.common.RequestTimingFilter : GET / -> 200 in 14 ms
INFO ... e.t.billable.common.RequestTimingFilter : GET /nothing-here -> 404 in 3 ms
```

```bash
git add .
git commit -m "feat(common): log duration of every request"
```

</details>
