# 03 — ORM basics: models and migrations · Spring Boot

[← Concepts](README.md) · Starting point: end of lesson 02 · Estimated time in class: 2 h · Independent work tasks: [README](README.md#independent-work-2-h)

## What we build today

- JPA and Flyway settings in `application.properties`, SQL logging in `application-dev.properties`
- Flyway migrations `V1__create_clients.sql` and `V2__create_projects.sql`, exactly as in
  [PROJECT.md](../../PROJECT.md#data-model)
- `project.BillingType` enum, stored as `hourly` / `fixed` / `capped` with `@EnumeratedValue`
- entities `client.Client` and `project.Project`, with `@ManyToOne` / `@OneToMany`
- `client.ClientRepository` and `project.ProjectRepository` with our first derived query methods
- `common.DevDataSeeder` — fixed development data, only in the `dev` profile
- the home page shows "3 clients · 7 projects"

**Final result:** starting the application on an empty database creates both tables (Flyway), checks that
the entities match them (Hibernate `validate`), inserts the test data (`dev` profile), and the home page at
`http://localhost:8080` shows the counts from the database.

---

## Step 1 — Check the dependencies and configure JPA

**Why:** Spring Boot 4 split its code into smaller modules. Tutorials written for Boot 3 miss some of
them, and the most painful one to miss is Flyway: without it, the application starts but never runs your
migrations.

Open `pom.xml` and check that these dependencies exist (Spring Initializr added them in lesson 01 when
you selected *Spring Data JPA*, *Flyway Migration* and *PostgreSQL Driver*):

```xml
<!-- pom.xml (inside <dependencies>) — check, do not duplicate -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-flyway</artifactId>
</dependency>
<dependency>
  <groupId>org.flywaydb</groupId>
  <artifactId>flyway-database-postgresql</artifactId>
</dependency>
<dependency>
  <groupId>org.postgresql</groupId>
  <artifactId>postgresql</artifactId>
  <scope>runtime</scope>
</dependency>
```

The Flyway starter may appear as `spring-boot-starter-flyway` or as the module `spring-boot-flyway` —
either is fine; what matters is that there is a Spring Boot Flyway artifact **and**
`flyway-database-postgresql`. If either is missing, add it without a `<version>` (Spring Boot's parent
chooses the version) and reload Maven in your IDE.

Now open `src/main/resources/application.properties`. From lesson 01 it already contains the datasource
lines and `spring.jpa.hibernate.ddl-auto=validate`. Keep them, and add the new lines below:

```properties
# src/main/resources/application.properties — add below the lines from lesson 01

# Do not keep the database session open while the view renders (explained in lesson 06).
spring.jpa.open-in-view=false
```

Then create a second file next to it, `src/main/resources/application-dev.properties`:

```properties
# src/main/resources/application-dev.properties

# Log every SQL statement Hibernate sends, and the values bound to the "?" parameters.
logging.level.org.hibernate.SQL=debug
logging.level.org.hibernate.orm.jdbc.bind=trace
```

**What happens:**

1. `ddl-auto=validate` (from lesson 01) — at startup Hibernate compares every entity with the real
   tables. Until today there were no entities, so there was nothing to check. If a table or column is
   missing, or has the wrong type, the application **refuses to start** and tells you which
   one. It never changes the schema. (`update` would change it silently and without history — never use
   it; see [Concepts §3](README.md#3-migrations-version-control-for-the-schema).)
2. `open-in-view=false` — by default Spring keeps the Hibernate session open until the page is rendered.
   That hides lazy-loading mistakes. We turn it off now, so mistakes show up early; lesson 06 explains
   the details.
3. `org.hibernate.SQL=debug` prints each SQL statement through the normal logger. You may see
   `spring.jpa.show-sql=true` in tutorials; it prints to standard output, bypassing the logging system —
   the logger setting is the better choice.
4. `org.hibernate.orm.jdbc.bind=trace` also prints the parameter values. It is noisy; turn it off when you
   do not need it.
5. `application-dev.properties` is read **on top of** `application.properties`, but only when the `dev`
   profile is active. Step 7 explains profiles; from then on you always start the app with `dev`. Tests
   and production do not use that profile, so their logs do not fill up with every SQL statement (and
   bound values, which may be personal data, never reach a production log).

The database URL is already there from lesson 01. When you run the application, **Docker Compose
support** also reads `compose.yaml` and starts the `postgres` service if it is not running.

---

## Step 2 — The first Flyway migrations

**Why:** Flyway runs SQL files from `src/main/resources/db/migration/` in version order, **once each**,
and records them in the table `flyway_schema_history`. The file name is the contract:

```
V1__create_clients.sql
│ │ └── description (underscores become spaces in the history table)
│ └──── two underscores separate version and description
└────── V = versioned migration; the number is the version
```

> Lesson 01 created only an empty `db/migration/.gitkeep`, so today's first file is `V1`. If you created
> a `V1` placeholder migration of your own in lesson 01, keep it (it has already run!) and number today's
> files from `V2`: `V2__create_clients.sql`, `V3__create_projects.sql`.

Create the folder `src/main/resources/db/migration` if it does not exist, and add:

```sql
-- src/main/resources/db/migration/V1__create_clients.sql
CREATE TABLE clients (
    id          BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    name        VARCHAR(120) NOT NULL,
    email       VARCHAR(255) NOT NULL,
    vat_number  VARCHAR(20),
    created_at  TIMESTAMP    NOT NULL,
    updated_at  TIMESTAMP    NOT NULL,
    CONSTRAINT clients_email_unique UNIQUE (email)
);
```

```sql
-- src/main/resources/db/migration/V2__create_projects.sql
CREATE TABLE projects (
    id                 BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    client_id          BIGINT       NOT NULL,
    name               VARCHAR(120) NOT NULL,
    billing_type       VARCHAR(20)  NOT NULL,
    hourly_rate_cents  BIGINT,
    fixed_price_cents  BIGINT,
    budget_cap_cents   BIGINT,
    rounding_minutes   SMALLINT     NOT NULL DEFAULT 1,
    archived_at        TIMESTAMP,
    created_at         TIMESTAMP    NOT NULL,
    updated_at         TIMESTAMP    NOT NULL,
    CONSTRAINT projects_client_id_foreign
        FOREIGN KEY (client_id) REFERENCES clients (id) ON DELETE RESTRICT
);

CREATE INDEX projects_client_id_index ON projects (client_id);
```

**What happens:**

1. `BIGINT GENERATED BY DEFAULT AS IDENTITY` — a 64-bit id that PostgreSQL generates for every new row.
   (This is the modern SQL-standard form of the older `BIGSERIAL`.) Hibernate's `GenerationType.IDENTITY`
   lets the database choose the id and reads it back after the insert.
2. Columns without `NOT NULL` accept `NULL`: `vat_number`, the three money columns and `archived_at`.
   Which money column is required depends on the billing type — validation checks that in lesson 05.
3. `CONSTRAINT clients_email_unique UNIQUE (email)` — the database refuses a second client with the same
   email, even if the application forgets to check. We give constraints explicit names (the same names
   Laravel generates), so error messages are readable and later migrations can refer to them.
4. `FOREIGN KEY ... ON DELETE RESTRICT` — a client who still has projects cannot be deleted.
5. `CREATE INDEX` — PostgreSQL does **not** index foreign keys automatically, and almost every project
   query filters by client. Lesson 14 covers indexes in depth.
6. Money is `BIGINT` cents (64-bit, so large totals never overflow), `billing_type` is a `VARCHAR` holding a lowercase string. See
   [Concepts §5](README.md#5-mapping-types) for why.

Do not start the application yet — without entities that is fine, but we want to see Flyway and
Hibernate validation working together in step 5.

---

## Step 3 — The `BillingType` enum

**Why:** in Java, enum constants are written in upper case (`HOURLY`), but the database stores lowercase
values (`hourly`), the same as the Laravel track. Plain `@Enumerated(EnumType.STRING)` stores the Java
name `HOURLY`. Since Jakarta Persistence 3.2 (Hibernate 7, which Spring Boot 4 uses), one annotation on
the enum chooses what is stored instead: **`@EnumeratedValue`**.

```java
// src/main/java/ee/ta25/billable/project/BillingType.java
package ee.ta25.billable.project;

import jakarta.persistence.EnumeratedValue;

/** How a project is billed. The database stores {@link #dbValue()}. */
public enum BillingType {
  HOURLY("hourly", "Hourly"),
  FIXED("fixed", "Fixed price"),
  CAPPED("capped", "Capped hourly");

  @EnumeratedValue private final String dbValue;
  private final String label;

  BillingType(String dbValue, String label) {
    this.dbValue = dbValue;
    this.label = label;
  }

  /** The value stored in {@code projects.billing_type}. */
  public String dbValue() {
    return dbValue;
  }

  /** Human-readable name for pages and forms. */
  public String label() {
    return label;
  }
}
```

**What happens:**

1. Each enum constant carries two values: the database value and a label. Enum constructors are always
   private, so nobody can create a fourth `BillingType` at runtime.
2. `@EnumeratedValue` marks the field whose value goes into the column. The rules: the field is `final`,
   never `null`, distinct for each constant, and a `String` (for `EnumType.STRING`). In step 5 the entity
   field gets `@Enumerated(EnumType.STRING)`; Hibernate then writes `"fixed"` for `FIXED` and reads
   `"fixed"` back as `FIXED`.
3. An unknown value in the database (say `weekly`) makes Hibernate throw when it reads the row — bad data
   fails loudly instead of turning into `null`.
4. Why not store the ordinal (`0`, `1`, `2`)? If someone reorders the constants, every existing row
   silently changes meaning. Strings do not have that problem.
5. Older tutorials (and many AI answers) write an `AttributeConverter<BillingType, String>` with
   `@Converter(autoApply = true)` and a `fromDbValue()` lookup loop for this. That still works and is
   the tool for conversions that are not a simple one-to-one value (for example a `Money` object to a
   number), but for enums `@EnumeratedValue` replaces the whole class.
6. The imports are `jakarta.persistence.*`. Old tutorials use `javax.persistence.*`, which does not exist
   in Spring Boot 3 or 4.

---

## Step 4 — The `Client` entity

**Why:** in the **Data Mapper** pattern the entity is a normal Java object that describes one row. It has
no `save()` method; a repository saves it (step 6).

```java
// src/main/java/ee/ta25/billable/client/Client.java
package ee.ta25.billable.client;

import ee.ta25.billable.project.Project;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.OneToMany;
import jakarta.persistence.Table;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

@Entity
@Table(name = "clients")
public class Client {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, length = 120)
  private String name;

  @Column(nullable = false, unique = true)
  private String email;

  @Column(name = "vat_number", length = 20)
  private String vatNumber;

  @OneToMany(mappedBy = "client")
  private List<Project> projects = new ArrayList<>();

  @CreationTimestamp
  @Column(name = "created_at", nullable = false, updatable = false)
  private LocalDateTime createdAt;

  @UpdateTimestamp
  @Column(name = "updated_at", nullable = false)
  private LocalDateTime updatedAt;

  /** Required by JPA. Application code uses the public constructor. */
  protected Client() {}

  public Client(String name, String email, String vatNumber) {
    this.name = name;
    this.email = email;
    this.vatNumber = vatNumber;
  }

  public Long getId() {
    return id;
  }

  public String getName() {
    return name;
  }

  public void setName(String name) {
    this.name = name;
  }

  public String getEmail() {
    return email;
  }

  public void setEmail(String email) {
    this.email = email;
  }

  public String getVatNumber() {
    return vatNumber;
  }

  public void setVatNumber(String vatNumber) {
    this.vatNumber = vatNumber;
  }

  /** Lazy: reading this list outside a transaction throws LazyInitializationException. */
  public List<Project> getProjects() {
    return projects;
  }

  public LocalDateTime getCreatedAt() {
    return createdAt;
  }

  public LocalDateTime getUpdatedAt() {
    return updatedAt;
  }
}
```

**What happens:**

1. `@Entity` tells Hibernate this class maps to a table. `@Table(name = "clients")` names the table —
   JPA would otherwise use the class name `client`, singular.
2. `@Id` + `@GeneratedValue(strategy = IDENTITY)` — the database generates the id. `id` is `null` until
   the entity is saved; that is why it is `Long` (an object) and not `long`.
3. `@Column(...)` describes the column. `nullable`, `length` and `unique` are **documentation** here: with
   `ddl-auto=validate`, Hibernate does not create constraints from them. The real constraints are in the
   Flyway files. Keep both in agreement.
4. Spring Boot turns camelCase field names into snake_case column names automatically (`vatNumber` →
   `vat_number`). We still write `name = "vat_number"` on multi-word columns, so the mapping is visible
   when you read the class.
5. `@OneToMany(mappedBy = "client")` — the inverse side of the relationship. `mappedBy` says: "the foreign
   key belongs to the field `client` in `Project`; this side only reads it". A `@OneToMany` is lazy by
   default: the list is loaded only when you touch it.
6. `@CreationTimestamp` / `@UpdateTimestamp` (from Hibernate) fill the timestamps on insert and update.
   `updatable = false` means Hibernate never changes `created_at` after the insert. They read the JVM's
   own clock and time zone, not a `Clock` bean (lesson 07), so a test cannot freeze them. If you ever
   need that, Spring Data JPA **auditing** does the same job with `@CreatedDate` / `@LastModifiedDate`
   and a `DateTimeProvider` that reads the `Clock`. We keep the simpler Hibernate annotations.
7. `protected Client()` — JPA needs a no-argument constructor to create objects when loading rows. We make
   it `protected` so application code must use the real constructor, which requires name and email.
8. There is no `setId()` and no setter for timestamps: nothing outside JPA should change them.

Why `LocalDateTime`: the column type is `TIMESTAMP` (without time zone), and `LocalDateTime` is the Java
type that matches it. Hibernate's validation checks that the types agree.

---

## Step 5 — The `Project` entity

```java
// src/main/java/ee/ta25/billable/project/Project.java
package ee.ta25.billable.project;

import ee.ta25.billable.client.Client;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.FetchType;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.JoinColumn;
import jakarta.persistence.ManyToOne;
import jakarta.persistence.Table;
import java.time.LocalDateTime;
import java.util.Set;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

@Entity
@Table(name = "projects")
public class Project {

  private static final Set<Integer> ALLOWED_ROUNDING = Set.of(1, 6, 15);

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @ManyToOne(fetch = FetchType.LAZY, optional = false)
  @JoinColumn(name = "client_id", nullable = false)
  private Client client;

  @Column(nullable = false, length = 120)
  private String name;

  @Enumerated(EnumType.STRING)
  @Column(name = "billing_type", nullable = false, length = 20)
  private BillingType billingType;

  @Column(name = "hourly_rate_cents")
  private Long hourlyRateCents;

  @Column(name = "fixed_price_cents")
  private Long fixedPriceCents;

  @Column(name = "budget_cap_cents")
  private Long budgetCapCents;

  @Column(name = "rounding_minutes", nullable = false)
  private short roundingMinutes = 1;

  @Column(name = "archived_at")
  private LocalDateTime archivedAt;

  @CreationTimestamp
  @Column(name = "created_at", nullable = false, updatable = false)
  private LocalDateTime createdAt;

  @UpdateTimestamp
  @Column(name = "updated_at", nullable = false)
  private LocalDateTime updatedAt;

  protected Project() {}

  public Project(Client client, String name, BillingType billingType, int roundingMinutes) {
    this.client = client;
    this.name = name;
    this.billingType = billingType;
    setRoundingMinutes(roundingMinutes);
  }

  public Long getId() {
    return id;
  }

  /** Lazy: only the id is available without a query. */
  public Client getClient() {
    return client;
  }

  public String getName() {
    return name;
  }

  public void setName(String name) {
    this.name = name;
  }

  public BillingType getBillingType() {
    return billingType;
  }

  public void setBillingType(BillingType billingType) {
    this.billingType = billingType;
  }

  public Long getHourlyRateCents() {
    return hourlyRateCents;
  }

  public void setHourlyRateCents(Long hourlyRateCents) {
    this.hourlyRateCents = hourlyRateCents;
  }

  public Long getFixedPriceCents() {
    return fixedPriceCents;
  }

  public void setFixedPriceCents(Long fixedPriceCents) {
    this.fixedPriceCents = fixedPriceCents;
  }

  public Long getBudgetCapCents() {
    return budgetCapCents;
  }

  public void setBudgetCapCents(Long budgetCapCents) {
    this.budgetCapCents = budgetCapCents;
  }

  public int getRoundingMinutes() {
    return roundingMinutes;
  }

  public void setRoundingMinutes(int roundingMinutes) {
    if (!ALLOWED_ROUNDING.contains(roundingMinutes)) {
      throw new IllegalArgumentException("Rounding must be 1, 6 or 15 minutes: " + roundingMinutes);
    }
    this.roundingMinutes = (short) roundingMinutes;
  }

  public LocalDateTime getArchivedAt() {
    return archivedAt;
  }

  public boolean isArchived() {
    return archivedAt != null;
  }

  public void archive(LocalDateTime when) {
    this.archivedAt = when;
  }

  public LocalDateTime getCreatedAt() {
    return createdAt;
  }

  public LocalDateTime getUpdatedAt() {
    return updatedAt;
  }
}
```

**What happens:**

1. `@ManyToOne(fetch = FetchType.LAZY, optional = false)` — the **owning side** of the relationship: this
   field holds the foreign key. The JPA default for `@ManyToOne` is **EAGER** (always load the client
   together with the project). That default is wrong for almost every application, so we always write
   `LAZY`. Hibernate then puts a *proxy* (a placeholder object) in the field; the real client is loaded
   only when you call a method like `getName()` on it.
2. `@JoinColumn(name = "client_id")` names the foreign-key column.
3. `@Enumerated(EnumType.STRING)` stores the enum as text; because `BillingType` has an `@EnumeratedValue`
   field (step 3), the text is `hourly`, not `HOURLY`. Without `@Enumerated` JPA would store the ordinal.
4. The money fields are `Long`, not `long`: they can be `null` (a fixed project has no hourly rate). The
   type matches the `BIGINT` column — `Integer` would fail validation (`expecting [integer]`, found
   `bigint`). When we calculate with them later (lessons 08 and 10) we unbox them to `long`.
5. `roundingMinutes` is a `short`, because the column is `SMALLINT` (2 bytes). With `ddl-auto=validate`,
   Hibernate checks that the Java type fits the column type; an `int` field would expect `INTEGER` and the
   application would not start. The getter still returns `int`, so the rest of the code does not care.
6. `setRoundingMinutes` checks rule R5 (1, 6 or 15). The entity protects its own data. In the independent
   work you add the same rule to the database as a `CHECK` constraint.
7. `archive(when)` instead of `setArchivedAt(...)`: the method name says what happens in the domain.

### Start the application

Now Flyway has something to create and Hibernate has something to validate:

```bash
./mvnw spring-boot:run
```

**What happens:**

1. Docker Compose support starts (or finds) the `postgres` container.
2. **Flyway** runs first: it creates `flyway_schema_history`, sees that versions 1 and 2 have not run, and
   runs both files.
3. **Hibernate** starts next and validates both entities against the real tables.
4. Tomcat starts on port 8080.

**Check it works:** the log contains lines like

```
Successfully validated 2 migrations
Migrating schema "public" to version "1 - create clients"
Migrating schema "public" to version "2 - create projects"
Successfully applied 2 migrations to schema "public", now at version v2
...
Tomcat started on port 8080 (http) with context path '/'
```

Look at the result in the database (reading is fine; changing by hand is not):

```bash
docker compose exec postgres psql -U billable -d billable -c '\d projects'
docker compose exec postgres psql -U billable -d billable -c 'SELECT version, description, success FROM flyway_schema_history'
```

The first command lists the 11 columns, the index and the foreign key. The second shows versions 1 and 2
with `success = t`.

Stop the application with `Ctrl+C`. Run the formatter, then commit:

```bash
./mvnw spotless:apply
```

**Commit:** `feat(db): create clients and projects tables with Client and Project entities`

---

## Step 6 — Repositories and derived queries

**Why:** a repository is the "mapper" of the Data Mapper pattern: the object that loads and saves
entities. With Spring Data you write only an **interface**; Spring creates the implementation when the
application starts.

```java
// src/main/java/ee/ta25/billable/client/ClientRepository.java
package ee.ta25.billable.client;

import java.util.List;
import java.util.Optional;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ClientRepository extends JpaRepository<Client, Long> {

  List<Client> findByNameContainingIgnoreCaseOrderByName(String text);

  Optional<Client> findByEmail(String email);
}
```

```java
// src/main/java/ee/ta25/billable/project/ProjectRepository.java
package ee.ta25.billable.project;

import java.util.List;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ProjectRepository extends JpaRepository<Project, Long> {

  long countByClientId(Long clientId);

  List<Project> findByClientIdOrderByName(Long clientId);
}
```

**What happens:**

1. `JpaRepository<Client, Long>` — a repository for `Client` entities whose id is a `Long`. You get
   `save`, `findById`, `findAll`, `count`, `deleteById` and more for free.
2. **Derived query methods**: Spring Data reads the method *name* and builds the query from it.
   `findByNameContainingIgnoreCaseOrderByName` means:

   | Part | Meaning | SQL |
   |---|---|---|
   | `findBy` | a query returning entities | `select ... from clients` |
   | `NameContaining` | the field `name` contains the argument | `where name like '%text%'` |
   | `IgnoreCase` | case-insensitive | `upper(name) like upper(?)` |
   | `OrderByName` | sort by `name` ascending | `order by name` |

3. `countByClientId` follows the path `client.id`: count projects whose client has this id. Because the
   id is stored in `projects.client_id`, no join is needed.
4. `findByEmail` returns `Optional<Client>`: the result may be missing, and the type forces the caller to
   handle that.
5. Spring checks every method name **at startup**. A typo like `findByNmae` stops the application with a
   clear error — much better than failing at runtime.

---

## Step 7 — Development data with `DevDataSeeder`

**Why:** we need the same fixed test data as everybody else (see
[Concepts §8](README.md#8-seeding-and-factories)). Why not put it in a Flyway migration? Because
migrations run in **every** environment — also in production and in the integration tests of lesson 14.
Fake clients must never reach production. A Flyway *repeatable* script (`R__seed.sql`) has the same
problem, and it is plain SQL that does not notice when the entities change.

So we use a small Java class that runs at startup **only when the `dev` profile is active**. It uses the
entities and repositories, so if the model changes, it stops compiling instead of silently inserting
wrong data.

```java
// src/main/java/ee/ta25/billable/common/DevDataSeeder.java
package ee.ta25.billable.common;

import ee.ta25.billable.client.Client;
import ee.ta25.billable.client.ClientRepository;
import ee.ta25.billable.project.BillingType;
import ee.ta25.billable.project.Project;
import ee.ta25.billable.project.ProjectRepository;
import java.time.LocalDateTime;
import java.time.ZoneId;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Profile;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

/**
 * Inserts fixed development data. Runs only with the "dev" profile, and only into an empty
 * database.
 */
@Component
@Profile("dev")
@Order(1)
class DevDataSeeder implements CommandLineRunner {

  private static final Logger log = LoggerFactory.getLogger(DevDataSeeder.class);
  private static final ZoneId TALLINN = ZoneId.of("Europe/Tallinn");

  private final ClientRepository clients;
  private final ProjectRepository projects;

  DevDataSeeder(ClientRepository clients, ProjectRepository projects) {
    this.clients = clients;
    this.projects = projects;
  }

  @Override
  public void run(String... args) {
    if (clients.count() > 0) {
      log.info("Development data already present, skipping seed");
      return;
    }

    Client acme = clients.save(new Client("Acme OÜ", "billing@acme.test", "EE101234567"));
    projects.save(hourly(acme, "Website redesign", 6500, 15));
    projects.save(capped(acme, "Support retainer", 6000, 200000, 6));
    Project intranet = hourly(acme, "Old intranet", 5000, 1);
    intranet.archive(LocalDateTime.now(TALLINN).minusMonths(3));
    projects.save(intranet);

    Client birch = clients.save(new Client("Birch & Daughters", "accounts@birch.test", null));
    projects.save(fixed(birch, "Logo and brand book", 150000, 1));
    projects.save(hourly(birch, "Webshop maintenance", 5500, 15));

    Client kala = clients.save(new Client("Kuressaare Kala AS", "arved@kala.test", "EE100987654"));
    projects.save(fixed(kala, "Stock app MVP", 480000, 15));
    projects.save(capped(kala, "Data migration", 7000, 350000, 6));

    log.info("Seeded {} clients and {} projects", clients.count(), projects.count());
  }

  private static Project hourly(Client client, String name, long rateCents, int rounding) {
    Project project = new Project(client, name, BillingType.HOURLY, rounding);
    project.setHourlyRateCents(rateCents);
    return project;
  }

  private static Project fixed(Client client, String name, long priceCents, int rounding) {
    Project project = new Project(client, name, BillingType.FIXED, rounding);
    project.setFixedPriceCents(priceCents);
    return project;
  }

  private static Project capped(
      Client client, String name, long rateCents, long capCents, int rounding) {
    Project project = new Project(client, name, BillingType.CAPPED, rounding);
    project.setHourlyRateCents(rateCents);
    project.setBudgetCapCents(capCents);
    return project;
  }
}
```

**What happens:**

1. `CommandLineRunner` — Spring Boot calls `run(...)` once, after the application has started.
2. `@Profile("dev")` — the bean exists only when the `dev` profile is active. Without it, the class is
   ignored completely.
3. `@Order(1)` — if there are several runners, this one runs first (we add a second one in step 8).
4. The class is package-private (no `public`): nothing else needs to use it. Spring can still create it.
5. `if (clients.count() > 0) return;` — the seeder runs on every start, so it must not insert the same
   clients twice (the unique email would stop it with an error anyway).
6. `clients.save(...)` returns the saved entity with its new id. We pass that `Client` into each
   `Project`; Hibernate writes its id into `client_id`.
7. The helper methods `hourly`, `fixed`, `capped` keep the data readable and make sure each project gets
   the right money fields for its type.
8. `LocalDateTime.now(TALLINN)` — as in lesson 01, we ask for the time in `Europe/Tallinn` explicitly.
   Plain `LocalDateTime.now()` uses the server's time zone, which is often UTC. (Lesson 07 replaces this
   with a `Clock` bean.)

Start the application **with the `dev` profile**:

```bash
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

In IntelliJ: *Run → Edit Configurations → your application → Active profiles: `dev`*. Or set the
environment variable `SPRING_PROFILES_ACTIVE=dev`.

We do **not** write `spring.profiles.active=dev` into `application.properties`: then every run — also
the tests in later lessons — would insert fake data.

**Check it works:** the log shows

```
The following 1 profile is active: "dev"
...
insert into clients (created_at,email,name,updated_at,vat_number) values (?,?,?,?,?) returning id
...
Seeded 3 clients and 7 projects
```

Stop and start again: now you see `Development data already present, skipping seed`.

**Commit:** `feat(db): add repositories and dev data seeder`

---

## Step 8 — First queries, and reading the SQL

**Why:** you must know which SQL your code runs. We add a **temporary** runner that calls a few
repository methods and prints the results next to the SQL in the log.

```java
// src/main/java/ee/ta25/billable/common/QueryPlayground.java
package ee.ta25.billable.common;

import ee.ta25.billable.client.Client;
import ee.ta25.billable.client.ClientRepository;
import ee.ta25.billable.project.ProjectRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Profile;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

/** TEMPORARY: try repository methods at startup. Delete before you finish lesson 03. */
@Component
@Profile("dev")
@Order(2)
class QueryPlayground implements CommandLineRunner {

  private static final Logger log = LoggerFactory.getLogger(QueryPlayground.class);

  private final ClientRepository clients;
  private final ProjectRepository projects;

  QueryPlayground(ClientRepository clients, ProjectRepository projects) {
    this.clients = clients;
    this.projects = projects;
  }

  @Override
  public void run(String... args) {
    log.info("Client count: {}", clients.count());

    var withA = clients.findByNameContainingIgnoreCaseOrderByName("a");
    log.info("Names containing 'a': {}", withA.stream().map(Client::getName).toList());

    Client acme = clients.findByEmail("billing@acme.test").orElseThrow();
    log.info("Acme has {} projects", projects.countByClientId(acme.getId()));

    var acmeProjects = projects.findByClientIdOrderByName(acme.getId());
    for (var p : acmeProjects) {
      log.info("  {} ({}), archived: {}", p.getName(), p.getBillingType().label(), p.isArchived());
    }
  }
}
```

Restart with the `dev` profile.

**Check it works:** in the log, each result appears after the SQL that produced it (shortened here):

```
select count(*) from clients c1_0
Client count: 3
select c1_0.id,c1_0.created_at,c1_0.email,c1_0.name,c1_0.updated_at,c1_0.vat_number from clients c1_0 where upper(c1_0.name) like upper(?) escape '\' order by c1_0.name
binding parameter (1:VARCHAR) <- [%a%]
Names containing 'a': [Acme OÜ, Birch & Daughters, Kuressaare Kala AS]
select c1_0.id,... from clients c1_0 where c1_0.email=?
binding parameter (1:VARCHAR) <- [billing@acme.test]
select count(p1_0.id) from projects p1_0 where p1_0.client_id=?
Acme has 3 projects
select p1_0.id,p1_0.archived_at,p1_0.billing_type,p1_0.budget_cap_cents,p1_0.client_id,... from projects p1_0 where p1_0.client_id=? order by p1_0.name
  Old intranet (Hourly), archived: true
  Support retainer (Capped hourly), archived: false
  Website redesign (Hourly), archived: false
```

**What happens:**

1. `c1_0`, `p1_0` are table aliases that Hibernate generates. They look strange, but the SQL is normal.
2. `Containing` became `like ?` with the value `%a%`. Spring Data also **escapes** `%` and `_` in your
   argument, so a user who searches for `50%` does not get a wildcard.
3. The values are **bound parameters** (`?`), shown in the `binding parameter` lines. They are never
   pasted into the SQL text — this is what protects you from SQL injection.
4. The project query selects `client_id`, but not the client itself: the `@ManyToOne` is lazy.

### An experiment: lazy loading outside a transaction

Add this line at the end of `run(...)` and restart:

```java
log.info("Acme projects via entity: {}", acme.getProjects().size());
```

The application fails to start with:

```
org.hibernate.LazyInitializationException: failed to lazily initialize a collection of role:
ee.ta25.billable.client.Client.projects: could not initialize proxy - no Session
```

**What happens:** `acme` was loaded by `findByEmail`. That repository call opened a database session,
ran the query and **closed the session**. The `projects` list was never loaded (it is lazy), and now there
is no session to load it with. This is the object–relational mismatch in action: following a reference
needs a query, and a query needs an open session. Lesson 06 shows the correct ways to load related data.
For now: when you need a client's projects, ask the `ProjectRepository`, as we did above. **Remove the
line again.**

Keep `QueryPlayground` while you do the independent practice queries, then delete it. Do not commit it.

---

## Step 9 — Real data on the home page

**Why:** a visible result, and the first controller that reads from the database.

Open `src/main/java/ee/ta25/billable/common/HomeController.java` from lesson 01. It has a `home` method
that puts `today` into the model, and (from the lesson 01 independent work) an `about` method. **Keep
both.** We add a constructor with the two repositories and two lines in `home`:

```java
// src/main/java/ee/ta25/billable/common/HomeController.java
package ee.ta25.billable.common;

import ee.ta25.billable.client.ClientRepository;
import ee.ta25.billable.project.ProjectRepository;
import java.time.LocalDate;
import java.time.ZoneId;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class HomeController {

  private static final ZoneId TALLINN = ZoneId.of("Europe/Tallinn");

  private final ClientRepository clients;
  private final ProjectRepository projects;

  HomeController(ClientRepository clients, ProjectRepository projects) {
    this.clients = clients;
    this.projects = projects;
  }

  @GetMapping("/")
  public String home(Model model) {
    model.addAttribute("today", LocalDate.now(TALLINN));
    model.addAttribute("clientCount", clients.count());
    model.addAttribute("projectCount", projects.count());
    return "home";
  }

  @GetMapping("/about")
  public String about() {
    return "about";
  }
}
```

In `src/main/resources/templates/home.html`, add this inside `<main>`, under the "Today is …" paragraph:

```html
<!-- src/main/resources/templates/home.html (inside <main>, under "Today is …") -->
<p>
  You have <strong th:text="${clientCount}">0</strong> clients
  and <strong th:text="${projectCount}">0</strong> projects.
</p>
```

**What happens:**

1. The controller has one constructor, so Spring injects both repositories without `@Autowired`
   (constructor injection — lesson 07 explains it in depth).
2. `count()` runs `select count(*)` — the database counts; Java receives one `long`.
3. `model.addAttribute` passes the numbers to the template; `th:text` prints them. The `0` inside the
   tags is only shown when you open the HTML file directly in a browser without Spring.
4. The view does not query anything itself — views display, controllers fetch (lesson 04).

**Check it works:** start with the `dev` profile, open `http://localhost:8080`. You see "You have **3**
clients and **7** projects." In the log you see the two `count(*)` queries for every page load.

```bash
./mvnw spotless:apply
./mvnw verify
```

**Commit:** `feat(home): show client and project counts`

---

## Troubleshooting

| Error / symptom | Cause | Fix |
|---|---|---|
| Rows saved as `HOURLY` (capitals), or reading fails with `No enum constant ee.ta25.billable.project.BillingType.hourly` | `@EnumeratedValue` missing on `BillingType.dbValue` | Add it (step 3); the field must be `final String` |
| `Schema-validation: wrong column type encountered in column [billing_type]` (expects a small integer) | `@Enumerated(EnumType.STRING)` missing on `Project.billingType`, so JPA maps the ordinal | Add it (step 5) |
| `Schema-validation: missing table [clients]` | Flyway did not run: Flyway module missing, or files not in `src/main/resources/db/migration` | Check step 1 dependencies and the folder name; look for "Migrating schema" in the log |
| `Wrong migration name format: V1_create_clients.sql` (or the file is silently ignored) | One underscore instead of two, or the file does not end in `.sql` | Name it `V1__create_clients.sql` |
| `Unsupported Database: PostgreSQL 17` | `flyway-database-postgresql` is missing | Add it (step 1) |
| `Validate failed: Migrations have failed validation. Migration checksum mismatch for migration version 1` | You edited a migration that has already run | Undo the edit and write a new migration. Only if the file never left your machine: reset the local database (below) |
| `Schema-validation: wrong column type encountered in column [rounding_minutes] in table [projects]; found [int2 (Types#SMALLINT)], but expecting [integer (Types#INTEGER)]` | The field is `int`/`Integer`, but the column is `SMALLINT` | Use `short` for the field (step 5) |
| `Schema-validation: missing column [vat_number] in table [clients]` | Field and column names disagree | Fix the `@Column(name = ...)` — or, if the column is really missing, write a new migration |
| `LazyInitializationException: ... could not initialize proxy - no Session` | You touched a lazy relation after the session closed | Load related data with a repository query (lesson 06 shows more options) |
| `No property 'nmae' found for type 'Client'` at startup | Typo in a derived query method name | Fix the method name; the names must match entity **field** names, not column names |
| Log says `No active profile set, falling back to 1 default profile: "default"` and no data appears | The `dev` profile is not active | `-Dspring-boot.run.profiles=dev`, or set it in the IDE run configuration |
| `duplicate key value violates unique constraint "clients_email_unique"` | Seeder ran without the `count() > 0` guard | Keep the guard; reset the local database |
| Startup fails with a Docker error such as *Docker is not running* | Docker Desktop is not started | Start Docker Desktop |

**Resetting your local development database.** Your data lives in a Docker volume. To start from zero:

```bash
docker compose down -v
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

`down -v` removes the container **and its data volume**. On the next start Flyway builds the schema from
V1 again, and the seeder inserts fresh data. This is the Spring equivalent of Laravel's
`migrate:fresh --seed`. Use it only on your own development database.

---

## Recap

- `application.properties`: `ddl-auto=validate`, `open-in-view=false`; `application-dev.properties`: SQL and
  bind-parameter logging.
- `db/migration/V1__create_clients.sql`, `V2__create_projects.sql` — the schema, owned by Flyway.
- `project.BillingType` with `@EnumeratedValue` — enum stored as lowercase string.
- `client.Client`, `project.Project` — entities with a lazy `@ManyToOne` and an inverse `@OneToMany(mappedBy)`.
- `client.ClientRepository`, `project.ProjectRepository` — Spring Data interfaces with derived queries.
- `common.DevDataSeeder` — fixed data, `dev` profile only, not a migration.
- `common.HomeController` + `home.html` — live counts on the home page.
- `QueryPlayground` — temporary; delete it after the practice queries.

---

## Independent work — solutions

> **The tasks are not here.** The task descriptions (Basic / Intermediate / Advanced, with acceptance
> criteria) are in this lesson's [README.md → Independent work (~2 h)](README.md#independent-work-2-h).
> Read them there first and build the features in your project. This section only contains the
> solutions — try each task yourself, then open a solution to compare or when you are stuck.

Reset your database first (see [Troubleshooting](#troubleshooting)) so the seed data is exactly as expected.

<details>
<summary><strong>Basic 1</strong> — Practice queries</summary>

Add these methods to the repositories. Q2 and Q7 use existing methods.

```java
// src/main/java/ee/ta25/billable/client/ClientRepository.java — add inside the interface
  List<Client> findAllByOrderByName(); // Q1

  List<Client> findByVatNumberIsNull(); // Q5
```

```java
// src/main/java/ee/ta25/billable/project/ProjectRepository.java — add inside the interface
  List<Project> findByBillingTypeOrderByName(BillingType billingType); // Q3

  List<Project> findByHourlyRateCentsGreaterThanEqualOrderByName(long cents); // Q4

  List<Project> findByArchivedAtIsNotNull(); // Q6

  Optional<Project> findFirstByBillingTypeOrderByFixedPriceCentsDesc(BillingType billingType); // Q8
```

(`ProjectRepository` needs `import java.util.Optional;`.)

Call them from `QueryPlayground.run(...)`:

```java
// src/main/java/ee/ta25/billable/common/QueryPlayground.java — replace the body of run(...)
    log.info("Q1 {}", clients.findAllByOrderByName().stream().map(Client::getName).toList());
    log.info("Q2 {}", projects.count());
    log.info(
        "Q3 {}",
        projects.findByBillingTypeOrderByName(BillingType.HOURLY).stream()
            .map(Project::getName)
            .toList());
    projects
        .findByHourlyRateCentsGreaterThanEqualOrderByName(6000)
        .forEach(p -> log.info("Q4 {} {}", p.getName(), p.getHourlyRateCents()));
    log.info("Q5 {}", clients.findByVatNumberIsNull().stream().map(Client::getName).toList());
    log.info("Q6 {}", projects.findByArchivedAtIsNotNull().stream().map(Project::getName).toList());
    Client acme = clients.findByEmail("billing@acme.test").orElseThrow();
    log.info("Q7 {}", projects.countByClientId(acme.getId()));
    projects
        .findFirstByBillingTypeOrderByFixedPriceCentsDesc(BillingType.FIXED)
        .ifPresent(p -> log.info("Q8 {} {}", p.getName(), p.getFixedPriceCents()));
```

(Add imports for `BillingType` and `Project` from `ee.ta25.billable.project`.)

The SQL in the log, shortened (Hibernate's aliases omitted):

| # | SQL |
|---|---|
| Q1 | `select ... from clients order by name` |
| Q2 | `select count(*) from projects` |
| Q3 | `select ... from projects where billing_type=? order by name` — bound value `hourly` (`@EnumeratedValue` at work) |
| Q4 | `select ... from projects where hourly_rate_cents>=? order by name` |
| Q5 | `select ... from clients where vat_number is null` |
| Q6 | `select ... from projects where archived_at is not null` |
| Q7 | `select ... from clients where email=?` then `select count(id) from projects where client_id=?` |
| Q8 | `select ... from projects where billing_type=? order by fixed_price_cents desc fetch first ? rows only` |

Notes:

- `findAllByOrderByName` — a derived query needs `By` even without a condition.
- Q4: fixed projects have `NULL` hourly rate; `NULL >= 6000` is not true in SQL, so they are not included.
- Q8: `findFirst...` adds a row limit in SQL. Loading all fixed projects and picking the maximum in Java
  would give the same answer, but the acceptance criteria forbid that.

</details>

<details>
<summary><strong>Basic 2</strong> — <code>Project</code> query methods</summary>

```java
// src/main/java/ee/ta25/billable/project/ProjectRepository.java
package ee.ta25.billable.project;

import java.util.List;
import java.util.Optional;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ProjectRepository extends JpaRepository<Project, Long> {

  long countByClientId(Long clientId);

  List<Project> findByClientIdOrderByName(Long clientId);

  /** Projects of one client that are not archived, sorted by name. */
  List<Project> findByClientIdAndArchivedAtIsNullOrderByName(Long clientId);

  /** All projects with the given billing type, sorted by name. */
  List<Project> findByBillingTypeOrderByName(BillingType billingType);

  // ...plus the practice-query methods from Basic 1, if you keep them
}
```

Check from `QueryPlayground`:

```java
    Client acme = clients.findByEmail("billing@acme.test").orElseThrow();
    log.info(
        "Active Acme: {}",
        projects.findByClientIdAndArchivedAtIsNullOrderByName(acme.getId()).stream()
            .map(Project::getName)
            .toList());
    // Active Acme: [Support retainer, Website redesign]
    log.info(
        "Capped: {}",
        projects.findByBillingTypeOrderByName(BillingType.CAPPED).stream()
            .map(Project::getName)
            .toList());
    // Capped: [Data migration, Support retainer]
```

SQL for the first one: `select ... from projects p1_0 where p1_0.client_id=? and p1_0.archived_at is null
order by p1_0.name`.

The parameter is `BillingType`, not `String`: `findByBillingTypeOrderByName("caped")` does not compile.
Hibernate turns `CAPPED` into `capped` for the query (the `@EnumeratedValue` field).

Derived names get long. When a name becomes hard to read (more than two or three conditions), use
`@Query` with JPQL instead:

```java
  @Query(
      "select p from Project p where p.client.id = :clientId and p.archivedAt is null"
          + " order by p.name")
  List<Project> findActiveByClient(@Param("clientId") Long clientId);
```

(Imports: `org.springframework.data.jpa.repository.Query`, `org.springframework.data.repository.query.Param`.)
JPQL uses **entity and field names** (`Project`, `archivedAt`), not table and column names.

</details>

<details>
<summary><strong>Intermediate</strong> — Check constraint on <code>rounding_minutes</code></summary>

A new migration — never an edit of `V2`:

```sql
-- src/main/resources/db/migration/V3__add_rounding_minutes_check.sql
ALTER TABLE projects
    ADD CONSTRAINT projects_rounding_minutes_check CHECK (rounding_minutes IN (1, 6, 15));
```

Restart the application. The log shows `Migrating schema "public" to version "3 - add rounding minutes check"`.

The later lessons number their example migrations for the in-class path, without this file (lesson 06
starts at `V3`). If you add it, every later migration simply takes the **next free** number — each
lesson reminds you.

**Test that it works.** The entity's `setRoundingMinutes` already refuses 5, so to test the *database*
rule we must go around the entity. Run this in `psql` — it is a test that the database rejects bad data;
it changes nothing:

```bash
docker compose exec postgres psql -U billable -d billable -c 'UPDATE projects SET rounding_minutes = 5 WHERE id = 1'
```

```
ERROR:  new row for relation "projects" violates check constraint "projects_rounding_minutes_check"
DETAIL:  Failing row contains (1, 1, Website redesign, hourly, 6500, null, null, 5, ...).
```

Why both the entity check **and** the constraint? The entity check gives a clear Java error early.
The constraint protects the data from everything else that can write to the table: a SQL script, a future
bulk update, another application. This is called *defence in depth*.

Flyway has no automatic "down" migration. If you need to remove the constraint later, that is a new
migration: `V4__drop_rounding_minutes_check.sql` with `ALTER TABLE projects DROP CONSTRAINT ...`.

</details>

<details>
<summary><strong>Advanced</strong> — When to use raw SQL (example answer)</summary>

Situations where raw SQL (a *native query*) is the better tool:

1. **Reports and aggregates** with several `GROUP BY` levels, `HAVING`, window functions or
   `generate_series` ("every week, including empty ones"). JPQL can express some of it, but not all, and
   the SQL is often easier to read and check.
2. **Bulk operations**: one `UPDATE ... WHERE ...` instead of loading thousands of entities and saving each.
3. **Database features JPA does not know**: check constraints, partial indexes, `FOR UPDATE SKIP LOCKED`,
   full-text search.
4. **Measured performance problems** where a hand-written query is clearly faster.

Example — new projects per week for the last 8 weeks, **including weeks with none**. The result is not
an entity, so we map it to a small record with Spring's `JdbcClient`:

```java
// src/main/java/ee/ta25/billable/project/WeeklyProjectCountQuery.java
package ee.ta25.billable.project;

import java.time.DayOfWeek;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.ZoneId;
import java.time.temporal.TemporalAdjusters;
import java.util.List;
import org.springframework.jdbc.core.simple.JdbcClient;
import org.springframework.stereotype.Component;

@Component
public class WeeklyProjectCountQuery {

  public record WeekCount(LocalDate weekStart, long newProjects) {}

  private static final ZoneId TALLINN = ZoneId.of("Europe/Tallinn");

  private final JdbcClient jdbc;

  WeeklyProjectCountQuery(JdbcClient jdbc) {
    this.jdbc = jdbc;
  }

  public List<WeekCount> lastWeeks(int weeks) {
    LocalDateTime to =
        LocalDate.now(TALLINN)
            .with(TemporalAdjusters.previousOrSame(DayOfWeek.MONDAY))
            .atStartOfDay();
    LocalDateTime from = to.minusWeeks(weeks - 1);
    return jdbc.sql(
            """
            SELECT CAST(w.week_start AS date) AS week_start,
                   COUNT(p.id)                AS new_projects
            FROM generate_series(CAST(:from AS timestamp), CAST(:to AS timestamp),
                                 interval '1 week') AS w(week_start)
            LEFT JOIN projects p
                   ON p.created_at >= w.week_start
                  AND p.created_at <  w.week_start + interval '1 week'
            GROUP BY w.week_start
            ORDER BY w.week_start
            """)
        .param("from", from)
        .param("to", to)
        .query(WeekCount.class)
        .list();
  }
}
```

- `generate_series` makes one row per Monday, even when no project was created that week. The
  `LEFT JOIN` keeps those rows, and `COUNT(p.id)` gives `0` for them. Standard JPQL cannot do this.
  (Hibernate 7's HQL has a `generate_series` function, but the plain SQL is shorter and you can check it
  directly in `psql`.)
- `:from` and `:to` are **named parameters**. The values go to PostgreSQL separately from the SQL text,
  so user input can never change the SQL itself.
- `CAST(:from AS timestamp)` tells PostgreSQL the type of the parameter, so it picks the `timestamp`
  version of `generate_series`.
- `.query(WeekCount.class)` maps the columns `week_start`, `new_projects` to the record components by
  name (snake_case to camelCase).
- Spring Boot creates the `JdbcClient` bean automatically because JDBC is on the classpath (it comes with
  Spring Data JPA).
- The alternative inside a repository is `@Query(value = "...", nativeQuery = true)`.

Expected result for `lastWeeks(8)` if you seeded this week: 8 rows; the last one (this week) has
`newProjects = 7`, the others `0`.

Downsides: the SQL is tied to PostgreSQL (`generate_series`); the IDE cannot rename columns inside the
string; the query is not checked at startup the way derived queries are. Keep raw SQL in one clearly named
class, and cover it with an integration test (lesson 14).

**When *not* to use raw SQL:** "number of projects and total budget cap per client" looks like a report,
but JPQL does it well with a **constructor projection** into a record:

```java
// src/main/java/ee/ta25/billable/client/ClientSummary.java
public record ClientSummary(String name, long projectCount, long totalCapCents) {}

// in ClientRepository
@Query(
    """
    SELECT new ee.ta25.billable.client.ClientSummary(
        c.name, COUNT(p), COALESCE(SUM(p.budgetCapCents), 0L))
    FROM Client c LEFT JOIN c.projects p
    GROUP BY c.id, c.name
    ORDER BY c.name
    """)
List<ClientSummary> summaries();
```

`SELECT new ...(...)` calls the record's constructor for every row. The class name must be written in
full. In Laravel the same report is `Client::withCount('projects')->withSum('projects', 'budget_cap_cents')`.

</details>
