# 11 — Unit testing · Spring Boot

[← Concepts](README.md) · Starting point: end of lesson 10 · Estimated time in class: 2 h · Independent work tasks: [README](README.md#independent-work-2-h)

---

## What we build today

- Unit tests in `src/test/java`, in the **same packages** as the classes they test — plain JUnit, no
  Spring context, no database.
- `common/MoneyTest.java` — grouped with `@Nested`: arithmetic, parsing euro strings (valid and invalid,
  with `assertThatThrownBy`), half-up rounding with basis points, comparison, formatting.
- `common/TimeRoundingTest.java` — a boundary table with `@ParameterizedTest` and `@CsvSource`.
- `project/ProjectTestData.java` — a small **test data builder** so tests can create a `Project` in one
  readable line.
- `billing/HourlyBillingTest.java`, `CappedHourlyBillingTest.java`, `FixedPriceBillingTest.java`.
- `common/DueDateCalculatorTest.java` — the weekend rule (R12) with `LocalDate`.
- One TDD cycle that adds `Money.sum()`.
- A deliberately bad test, and why it is bad.
- A JaCoCo coverage report.

**Final result:** `./mvnw test` runs about 65 test cases. The unit tests themselves take well under a
second; IntelliJ shows them as a tree of readable sentences. CI (`./mvnw verify`, from lesson 02) runs
them on every push without any change.

### The lesson 10 code we test

Tests are written against the **public behaviour** of a class. This is the behaviour this guide expects
from lesson 10. If your method names differ slightly, change the tests, not the lesson 10 code — unless
a test finds a real bug.

| Class | Behaviour the tests rely on |
|---|---|
| `common.Money` | `record Money(long cents)`; `Money.zero()`, `Money.fromCents(long)`, `Money.fromEuros(String)` (accepts `"85.50"`, `"85,5"`, `"1200"`, `"-3.10"`, ignores surrounding spaces; throws `IllegalArgumentException` for more than two decimals or anything else), `add`, `subtract`, `multiply(long)`, `percentageBp(int)` (half up, away from zero), `min`, `max`, `isZero()`, `isNegative()`, `greaterThanOrEqual(Money)`, `compareTo`, `format()` (Estonian style `"1 234,50 €"`) |
| `common.TimeRounding` | `static long roundUp(long minutes, int increment)`; throws `IllegalArgumentException` for negative minutes or an increment other than 1, 6 or 15 |
| `billing.*` | `Money calculate(Project project, long billableMinutes, Money alreadyBilled)` on `HourlyBilling`, `CappedHourlyBilling` (constructor takes a `HourlyBilling`, lesson 08), `FixedPriceBilling`; `HourlyBilling.forMinutes(Money rate, long minutes)` throws `IllegalArgumentException` for negative minutes |
| `common.DueDateCalculator` | `LocalDate dueDate(LocalDate issuedOn)` (R12) |
| `project.Project` | constructor `Project(Client client, String name, BillingType billingType, int roundingMinutes)` (lesson 03) and setters for `hourlyRateCents`, `fixedPriceCents`, `budgetCapCents` (`Long`, nullable — the columns are `bigint`) and `roundingMinutes` (`int`) — only `ProjectTestData` (step 5) depends on this |

---

## Step 1 — Look at the test setup you already have

**Why:** know where tests live, which libraries you have and how Maven finds them before writing any.

Open `pom.xml`. Spring Initializr added test dependencies for you. In Spring Boot 4 there is one test
starter per feature starter, for example:

```xml
<!-- pom.xml (excerpt — already there, do not type) -->
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

1. `spring-boot-starter-test` brings **JUnit Jupiter** (the test framework), **AssertJ** (fluent
   assertions), **Mockito** (lesson 12) and Spring's test support. You do not choose versions — the Spring
   Boot parent manages them.
2. `<scope>test</scope>` means: available in `src/test/java`, not packaged into the application jar.
3. **JUnit version note.** The API we use is known as "JUnit 5" (package `org.junit.jupiter.api`). Spring
   Boot 4 actually ships **JUnit 6**, which keeps the same packages and annotations — everything in this
   guide works unchanged. What does *not* work: JUnit **4** code from old tutorials (`org.junit.Test`,
   `@Before`, `@RunWith`). If you see those imports, the example is too old.
4. Maven's **Surefire** plugin runs every class in `src/test/java` whose name ends in `Test` (or starts
   with `Test`, or ends in `Tests`) during the `test` phase.

Open `src/test/java/ee/ta25/billable/BillableApplicationTests.java`. It has one test, `contextLoads()`,
annotated with `@SpringBootTest`. That test starts the **whole application**: all beans, JPA, Flyway, the
database connection. It is useful as a smoke test, but it is the opposite of a unit test: it needs
Docker for its PostgreSQL container (lesson 01, Step 9) and takes seconds. We leave it alone today.

Run the tests that exist:

```bash
./mvnw test
```

**Check it works:** `BUILD SUCCESS` and `Tests run: 1, Failures: 0, Errors: 0`. If `contextLoads` fails
with `Could not find a valid Docker environment`, start Docker Desktop: Testcontainers needs it to start
the test database.

---

## Step 2 — The first test: adding money

**Why:** the smallest possible test shows the whole cycle: write, run, fail, pass.

Test classes go in the **same package** as the class under test, under `src/test/java`. For
`src/main/java/ee/ta25/billable/common/Money.java` that is:

```java
// src/test/java/ee/ta25/billable/common/MoneyTest.java
package ee.ta25.billable.common;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

class MoneyTest {

  @Test
  void adds_two_amounts() {
    // Arrange
    var price = Money.fromCents(1250);
    var extra = Money.fromCents(399);

    // Act
    var total = price.add(extra);

    // Assert
    assertThat(total).isEqualTo(Money.fromCents(1649));
  }
}
```

**What happens:**

1. **Same package** means the test can see package-private members, and IntelliJ shows the test next to
   the class ("Navigate → Test", `Ctrl+Shift+T` / `Cmd+Shift+T`).
2. The class and the method are **package-private** (no `public`). JUnit 5 does not need `public`, and
   leaving it out keeps the code short.
3. There is **no Spring annotation** on the class. No `@SpringBootTest`, no `@ExtendWith`. JUnit creates
   the test object with `new MoneyTest()` and calls the method. That is what makes it fast.
4. `@Test` marks the method as a test. The name uses underscores so it reads as a sentence: *"Money adds
   two amounts."* Google Java Style allows underscores in test method names.
   (Lesson 02's `CODESTYLE.md` left the test-name style to a group vote, with a lowerCamelCase example.
   This guide uses underscores from here on; follow what your group decided and record it in
   `CODESTYLE.md`.)
5. `assertThat(actual).isEqualTo(expected)` reads left to right: "assert that total is equal to 16.49 €".
   `Money` is a **record**, and records get `equals()` from their fields, so two `Money` objects with 1649
   cents are equal.

Run it — in IntelliJ with the green arrow next to the class, or from the terminal:

```bash
./mvnw test -Dtest=MoneyTest
```

**Check it works:** `Tests run: 1, Failures: 0`.

Now **make it fail on purpose** — this proves the test can see a bug. In `Money.add()`, change `+` to `-`
and run again:

```text
[ERROR] Failures:
[ERROR]   MoneyTest.adds_two_amounts:20
expected: Money[cents=1649]
 but was: Money[cents=851]
```

AssertJ prints both values using the record's `toString()`. Undo the change; the test is green again.
Every time you write a new kind of test, break the code once and watch it go red.

---

## Step 3 — The rest of `MoneyTest`, grouped with `@Nested`

**Why:** `Money` has about ten behaviours. `@Nested` groups them by topic, so the test report reads like a
table of contents: *Money → Parsing euros → rejects "12,50"*.

Replace the whole file:

```java
// src/test/java/ee/ta25/billable/common/MoneyTest.java
package ee.ta25.billable.common;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import org.junit.jupiter.params.provider.ValueSource;

class MoneyTest {

  @Nested
  class Arithmetic {

    @Test
    void adds_two_amounts() {
      assertThat(Money.fromCents(1250).add(Money.fromCents(399))).isEqualTo(Money.fromCents(1649));
    }

    @Test
    void subtracts_an_amount() {
      assertThat(Money.fromCents(1000).subtract(Money.fromCents(250))).isEqualTo(Money.fromCents(750));
    }

    @Test
    void multiplies_by_a_whole_number() {
      assertThat(Money.fromCents(1250).multiply(3)).isEqualTo(Money.fromCents(3750));
    }
  }

  @Nested
  class ParsingEuros {

    @ParameterizedTest(name = "\"{0}\" is {1} cents")
    @CsvSource(
        textBlock =
            """
            12,         1200
            12.5,       1250
            12.50,      1250
            # a comma inside a value needs quotes
            '12,50',    1250
            0.01,       1
            0,          0
            1234.56,    123456
            -3.10,      -310
            # quotes keep the spaces, which fromEuros must ignore
            ' 12.50 ',  1250
            """)
    void parses_a_valid_euro_string(String input, long expectedCents) {
      assertThat(Money.fromEuros(input)).isEqualTo(Money.fromCents(expectedCents));
    }

    @ParameterizedTest(name = "rejects \"{0}\"")
    @ValueSource(strings = {"", "abc", "12.345", "12.", "1.234,56"})
    void rejects_an_invalid_euro_string(String input) {
      assertThatThrownBy(() -> Money.fromEuros(input))
          .isInstanceOf(IllegalArgumentException.class);
    }
  }

  @Nested
  class Percentages {

    @ParameterizedTest(name = "{0} cents x {1} bp = {2} cents")
    @CsvSource(
        textBlock =
            """
            # 24 % of 12.50 is exactly 3.00
            1250, 2400, 300
            # 24 % of 10.21 is 2.4504 -> rounds down
            1021, 2400, 245
            # 24 % of 10.23 is 2.4552 -> rounds up
            1023, 2400, 246
            # 50 % of 1 cent is exactly half -> rounds up
            1, 5000, 1
            # 50 % of 3 cents is 1.5 -> rounds up
            3, 5000, 2
            # 49.99 % of 1 cent is just below half -> rounds down
            1, 4999, 0
            # 24 % of nothing
            0, 2400, 0
            # negative half rounds away from zero
            -1, 5000, -1
            # negative 1.5 rounds away from zero
            -3, 5000, -2
            """)
    void takes_a_percentage_rounded_half_up(long cents, int basisPoints, long expectedCents) {
      assertThat(Money.fromCents(cents).percentageBp(basisPoints))
          .isEqualTo(Money.fromCents(expectedCents));
    }
  }

  @Nested
  class Comparing {

    @Test
    void knows_when_it_is_zero() {
      assertThat(Money.fromCents(0).isZero()).isTrue();
      assertThat(Money.fromCents(1).isZero()).isFalse();
    }

    @Test
    void a_result_below_zero_is_negative() {
      assertThat(Money.fromCents(100).subtract(Money.fromCents(250)).isNegative()).isTrue();
      assertThat(Money.fromCents(0).isNegative()).isFalse();
    }

    @Test
    void an_equal_amount_counts_as_greater_or_equal() {
      assertThat(Money.fromCents(500).greaterThanOrEqual(Money.fromCents(500))).isTrue();
    }

    @Test
    void a_smaller_amount_is_not_greater_or_equal() {
      assertThat(Money.fromCents(499).greaterThanOrEqual(Money.fromCents(500))).isFalse();
    }
  }

  @Nested
  class Formatting {

    @ParameterizedTest(name = "{0} cents -> \"{1}\"")
    @CsvSource(
        delimiter = '|',
        textBlock =
            """
            1250   | 12,50 €
            5      | 0,05 €
            0      | 0,00 €
            123456 | 1 234,56 €
            -5     | -0,05 €
            """)
    void formats_as_euros(long cents, String expected) {
      assertThat(Money.fromCents(cents).format()).isEqualTo(expected);
    }
  }
}
```

**What happens:**

1. `@Nested` marks an **inner class** (not `static`) as a group of tests. JUnit runs the tests of each
   group and shows them as a tree. Nested classes can have their own setup; we do not need it here.
2. `@ParameterizedTest` replaces `@Test` when the method takes arguments. `@CsvSource` gives the rows:
   each line is one row, values separated by commas. A value that itself contains a comma (`'12,50'`) or
   spaces that must be kept (`' 12.50 '`) goes in single quotes. JUnit **converts** each value to the parameter
   type — `"1250"` to `long`, `"12.50"` stays a `String`.
3. `name = "\"{0}\" is {1} cents"` controls the display name of each row. `{0}` is the first argument.
   When a row fails, the report shows `"12.345" …` instead of `[3]`.
4. `@ValueSource(strings = {...})` is the simpler source for a single parameter. The empty string `""`
   is a row too.
5. `assertThatThrownBy(() -> ...)` takes a **lambda** — a small piece of code that AssertJ runs itself.
   It passes only if the lambda throws, and then checks the type with `isInstanceOf`. If `fromEuros()`
   returns normally, the failure is `Expecting code to raise a throwable.`
6. `textBlock = """ ... """` is a Java **text block** (multi-line string). Inside a `@CsvSource` text
   block, lines starting with `#` are comments. We use them to explain each row — the table *is* the
   documentation of R3. The two negative rows check the lesson 10 decision that "half up" means *away
   from zero* (−0.5 → −1).
7. With 24 % VAT you can **never** hit exactly half a cent: `cents × 2400` would have to end in `5000`,
   which needs `cents × 24` to end in `50`, and `cents × 24 = 100k + 50` has no whole-number solution
   because the left side is a multiple of 4 while `100k + 50` never is. So the "exactly half" rows use 50 %.
   Choosing test rows often teaches you something about the rule.
8. The formatting rows use `delimiter = '|'` because the expected text contains a comma (Estonian
   style, `12,50 €`) and spaces. Unquoted values are trimmed at both ends, but spaces *inside* a value
   (`1 234,56 €`) are kept. If your `format()` produces another style, change the expected strings — the
   test documents **your** decision.
9. There is no "does not change the original" test as in the Laravel track: a record's fields are
   `final`, so the compiler already guarantees immutability.

**Check it works:**

```bash
./mvnw test -Dtest='MoneyTest*'
```

The quotes and `*` make sure the nested classes (`MoneyTest$Arithmetic`, …) are included.
Result: `Tests run: 35, Failures: 0`. In IntelliJ the run window shows the tree:

```text
MoneyTest
├─ Arithmetic
│  ├─ adds_two_amounts()
│  └─ ...
├─ ParsingEuros
│  ├─ parses_a_valid_euro_string(String, long)
│  │  ├─ "12" is 1200 cents
│  │  └─ ...
│  └─ rejects_an_invalid_euro_string(String)
│     ├─ rejects "1.234,56"
...
```

If a parsing row fails, first decide who is right. Maybe your lesson 10 code does not accept a comma. That
is a design decision — make it on purpose, and move the row to the table that matches it.

Before committing, format the code — Spotless also checks `src/test`:

```bash
./mvnw spotless:apply
```

**Commit:** `test(money): cover arithmetic, parsing, rounding and formatting`

---

## Step 4 — `TimeRoundingTest`: a boundary table

**Why:** R5 (round **up** to 1, 6 or 15 minutes) is the classic place for off-by-one bugs.

```java
// src/test/java/ee/ta25/billable/common/TimeRoundingTest.java
package ee.ta25.billable.common;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

class TimeRoundingTest {

  @ParameterizedTest(name = "{0} min with {1}-min rounding -> {2} min")
  @CsvSource(
      textBlock =
          """
          # 15-minute blocks: the boundary at 15
          0, 15, 0
          1, 15, 15
          14, 15, 15
          15, 15, 15
          16, 15, 30
          720, 15, 720
          # 6-minute blocks: the boundary at 6
          5, 6, 6
          6, 6, 6
          7, 6, 12
          # 1-minute blocks: nothing changes
          7, 1, 7
          0, 1, 0
          """)
  void rounds_minutes_up_to_the_increment(long minutes, int increment, long expected) {
    assertThat(TimeRounding.roundUp(minutes, increment)).isEqualTo(expected);
  }

  @ParameterizedTest(name = "rejects {0} min with {1}-min rounding")
  @CsvSource({"-1, 15", "10, 0", "10, 7"})
  void rejects_invalid_input(long minutes, int increment) {
    assertThatThrownBy(() -> TimeRounding.roundUp(minutes, increment))
        .isInstanceOf(IllegalArgumentException.class);
  }
}
```

**What happens:**

1. For each increment we test **below, on and above** the first boundary. We do not repeat it for 29,
   30, 31: the same arithmetic handles every block, so one boundary per increment is one equivalence
   class.
2. The row `0, 15, 0` finds the most common bug. A formula like `(minutes / increment + 1) * increment`
   passes 1, 14 and 16, but turns 0 into 15 and 15 into 30.
3. `720` minutes is 12 hours — the maximum length of an entry (R4), a realistic large value.
4. The invalid rows: negative minutes, increment 0 (would divide by zero), increment 7 (not allowed by
   the rules).

**Check it works:** `./mvnw test -Dtest=TimeRoundingTest` → `Tests run: 14`.

Lesson 10's `roundUp()` validates its input, so these rows should pass at once. Prove that they really
test something: comment out the increment check in `TimeRounding.roundUp()` and run again. The row
`10, 7` goes red, and `10, 0` fails with an `ArithmeticException` (`/ by zero`) instead of the expected
exception. Undo the change. If the rows fail *before* you changed anything, your `roundUp()` is missing
the guard clauses — add them:

```java
// src/main/java/ee/ta25/billable/common/TimeRounding.java — first lines inside roundUp()
    if (increment != 1 && increment != 6 && increment != 15) {
      throw new IllegalArgumentException("Rounding must be 1, 6 or 15 minutes, got " + increment);
    }
    if (minutes < 0) {
      throw new IllegalArgumentException("Minutes must not be negative, got " + minutes);
    }
```

**Commit:** `test(time-rounding): boundary table for 1, 6 and 15 minute rounding`

---

## Step 5 — A test data builder for `Project`

**Why:** the strategies take a `Project` entity. A JPA entity is still a normal Java object — `new
Project(...)` does not touch the database. Only a repository does. So we can create projects in memory.

But a `Project` needs a client, a name, a billing type and three optional amounts. If every test calls
the constructor and four setters, the tests become long, and **every** test breaks when the constructor
changes. A **test data builder** solves both: tests say only what matters to them, and the constructor
call lives in one place.

```java
// src/test/java/ee/ta25/billable/project/ProjectTestData.java
package ee.ta25.billable.project;

import ee.ta25.billable.client.Client;

/**
 * Builds {@link Project} objects for unit tests, without a database. Every value has a sensible
 * default, so a test only sets what it is about.
 */
public final class ProjectTestData {

  private String name = "Website redesign";
  private BillingType billingType = BillingType.HOURLY;
  private Long hourlyRateCents = 6000L;
  private Long fixedPriceCents;
  private Long budgetCapCents;
  private int roundingMinutes = 1;

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

  /** The only place in the tests that knows how a Project is constructed. */
  public Project build() {
    var client = new Client("Test client OÜ", "client@example.com", null);
    var project = new Project(client, name, billingType, roundingMinutes);
    project.setHourlyRateCents(hourlyRateCents);
    project.setFixedPriceCents(fixedPriceCents);
    project.setBudgetCapCents(budgetCapCents);
    return project;
  }
}
```

**What happens:**

1. The class is in `src/test/java`, so it is never part of the application. It is in package
   `ee.ta25.billable.project`, next to `Project`, and `public` so tests in other packages (`billing`) can
   use it.
2. The static methods `hourly(...)`, `capped(...)`, `fixed(...)` are the three ways a test starts. They
   set only what defines that billing type.
3. `named(...)` and `rounding(...)` return `this`, so calls can be chained:
   `ProjectTestData.capped(6000, 100000).rounding(15).build()`. That style is called a **fluent**
   interface.
4. The cent fields are `Long` (object, may be `null`) because the columns are nullable `bigint`: a fixed
   project has no hourly rate. The factory methods take a primitive `long`, so a test can write
   `hourly(6000)` and Java widens the `int` literal automatically.
5. `build()` creates the entity. `new Project(...)` and the setters are plain Java — Hibernate is not
   involved until a repository saves the object. The client is created in memory as well.
6. **Adapt `build()` to your entity.** If your `Project` constructor takes all values at once, or your
   `Client` constructor has other parameters, change only these lines. That is the whole point: when the
   entity changes, one method changes, not thirty tests.

**Check it works:** `./mvnw test-compile` → `BUILD SUCCESS`. If it fails with `cannot find symbol`, your
constructor or setter names differ — fix `build()`.

---

## Step 6 — Tests for the three billing strategies

### 6a — `HourlyBillingTest`

```java
// src/test/java/ee/ta25/billable/billing/HourlyBillingTest.java
package ee.ta25.billable.billing;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import ee.ta25.billable.common.Money;
import ee.ta25.billable.project.ProjectTestData;
import org.junit.jupiter.api.Test;

class HourlyBillingTest {

  private final HourlyBilling strategy = new HourlyBilling();

  @Test
  void bills_ninety_minutes_at_sixty_euros_per_hour() {
    var project = ProjectTestData.hourly(6000).build();

    var amount = strategy.calculate(project, 90, Money.fromCents(0));

    assertThat(amount).isEqualTo(Money.fromCents(9000));
  }

  @Test
  void bills_nothing_for_zero_minutes() {
    var project = ProjectTestData.hourly(6000).build();

    var amount = strategy.calculate(project, 0, Money.fromCents(0));

    assertThat(amount).isEqualTo(Money.fromCents(0));
  }

  @Test
  void rejects_negative_minutes() {
    var project = ProjectTestData.hourly(6000).build();

    assertThatThrownBy(() -> strategy.calculate(project, -1, Money.zero()))
        .isInstanceOf(IllegalArgumentException.class);
  }

  @Test
  void rounds_exactly_half_a_cent_up() {
    // 90.30 € per hour for 1 minute = 150.5 cents
    var project = ProjectTestData.hourly(9030).build();

    var amount = strategy.calculate(project, 1, Money.fromCents(0));

    assertThat(amount).isEqualTo(Money.fromCents(151));
  }
}
```

**What happens:**

1. `private final HourlyBilling strategy = new HourlyBilling();` — JUnit creates a **new instance of the
   test class for every test method**, so every test gets a fresh strategy. Nothing leaks between tests.
2. We create the strategy with `new`, although in the application Spring creates it (`@Component`). A
   unit test does not need Spring to build an object that has no dependencies.
3. `rejects_negative_minutes` covers the guard clause in `HourlyBilling.forMinutes()`. Negative minutes
   can only come from a bug elsewhere; failing loudly is the lesson 10 rule.
4. `rounds_exactly_half_a_cent_up` uses a "strange" rate on purpose: 9030 ÷ 60 = 150.5 cents. That is the
   R3/R6 half-up boundary. The comment explains the number.

### 6b — `CappedHourlyBillingTest`

```java
// src/test/java/ee/ta25/billable/billing/CappedHourlyBillingTest.java
package ee.ta25.billable.billing;

import static org.assertj.core.api.Assertions.assertThat;

import ee.ta25.billable.common.Money;
import ee.ta25.billable.project.Project;
import ee.ta25.billable.project.ProjectTestData;
import org.junit.jupiter.api.Test;

class CappedHourlyBillingTest {

  // 60 €/h, cap 1000 €. 600 minutes = 600 € at the hourly rate.
  private final Project project = ProjectTestData.capped(6000, 100_000).build();
  private final CappedHourlyBilling strategy = new CappedHourlyBilling(new HourlyBilling());

  @Test
  void bills_the_hourly_amount_while_far_below_the_cap() {
    assertThat(strategy.calculate(project, 600, Money.fromCents(0))).isEqualTo(Money.fromCents(60_000));
  }

  @Test
  void bills_nothing_when_the_cap_is_already_reached() {
    assertThat(strategy.calculate(project, 600, Money.fromCents(100_000)))
        .isEqualTo(Money.fromCents(0));
  }

  @Test
  void never_bills_a_negative_amount_when_already_over_the_cap() {
    assertThat(strategy.calculate(project, 600, Money.fromCents(120_000)))
        .isEqualTo(Money.fromCents(0));
  }
}
```

**What happens:**

1. `CappedHourlyBilling` receives a `HourlyBilling` through its constructor (lesson 08). In the test we
   pass a **real** `new HourlyBilling()`, not a mock: it is pure and fast, and the capped rule really does
   depend on the hourly amount. Doing by hand what Spring does at startup is all dependency injection is.
2. `100_000` — Java allows underscores in number literals. It makes cent amounts readable: 1000.00 €.
3. "Already over the cap" can happen when the owner lowers the cap after billing. The `max(…, 0)` in the
   strategy must stop a negative invoice line; this test guards it.
4. The central boundary — remaining budget **just below, equal to, just above** the hourly amount — is
   part of Task 1. Try it yourself first.

### 6c — `FixedPriceBillingTest`

```java
// src/test/java/ee/ta25/billable/billing/FixedPriceBillingTest.java
package ee.ta25.billable.billing;

import static org.assertj.core.api.Assertions.assertThat;

import ee.ta25.billable.common.Money;
import ee.ta25.billable.project.Project;
import ee.ta25.billable.project.ProjectTestData;
import org.junit.jupiter.api.Test;

class FixedPriceBillingTest {

  private final Project project = ProjectTestData.fixed(250_000).build();
  private final FixedPriceBilling strategy = new FixedPriceBilling();

  @Test
  void bills_the_full_price_when_nothing_was_billed_yet() {
    assertThat(strategy.calculate(project, 600, Money.fromCents(0)))
        .isEqualTo(Money.fromCents(250_000));
  }

  @Test
  void bills_only_the_remaining_part_after_a_partial_invoice() {
    assertThat(strategy.calculate(project, 600, Money.fromCents(100_000)))
        .isEqualTo(Money.fromCents(150_000));
  }
}
```

**Check it works:**

```bash
./mvnw test -Dtest='*BillingTest'
```

→ `Tests run: 9, Failures: 0`.

**Commit:** `test(billing): unit tests for hourly, capped and fixed-price strategies`

---

## Step 7 — `DueDateCalculatorTest`: dates without the clock

**Why:** R12 is a date rule. `DueDateCalculator` receives the issue date as an argument, so the test
decides "today" — no `LocalDate.now()`, no dependence on the day you run it.

`DueDateCalculator` is the lesson 10 independent task (Task 1). If you skipped it, create
`common/DueDateCalculator.java` from that solution first — lesson 13 uses it too.

```java
// src/test/java/ee/ta25/billable/common/DueDateCalculatorTest.java
package ee.ta25.billable.common;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.LocalDate;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

class DueDateCalculatorTest {

  private final DueDateCalculator calculator = new DueDateCalculator();

  @ParameterizedTest(name = "issued {0} -> due {1}")
  @CsvSource(
      textBlock =
          """
          # due on a Friday stays on Friday
          2026-10-02, 2026-10-16
          # due on a Saturday moves to Monday
          2026-10-03, 2026-10-19
          # due on a Sunday moves to Monday
          2026-10-04, 2026-10-19
          # due on a Monday stays on Monday
          2026-10-05, 2026-10-19
          # moving across the new year
          2026-12-19, 2027-01-04
          """)
  void due_date_is_fourteen_days_later_but_never_on_a_weekend(
      LocalDate issuedOn, LocalDate expectedDueOn) {
    assertThat(calculator.dueDate(issuedOn)).isEqualTo(expectedDueOn);
  }
}
```

**What happens:**

1. JUnit converts `"2026-10-02"` to a `LocalDate` automatically (ISO format). No parsing code in the test.
2. `LocalDate` has no time and no time zone, so the comparison checks exactly the rule.
3. 14 days are two weeks, so the due date falls on the **same weekday** as the issue date. That is how the
   rows were chosen: Friday must not move; Saturday moves +2; Sunday moves +1. An implementation that
   always adds 2 days on weekends passes the Saturday row and fails the Sunday row.
4. The last row crosses a month and a year.

**Check it works:** `./mvnw test -Dtest=DueDateCalculatorTest` → `Tests run: 5`.

**Commit:** `test(due-date): weekend rule as a parameterized test`

---

## Step 8 — Running tests: the commands you will use

| Command | What it does |
|---|---|
| `./mvnw test` | Compile and run all tests |
| `./mvnw test -Dtest=TimeRoundingTest` | One class |
| `./mvnw test -Dtest='MoneyTest*'` | One class including its `@Nested` classes |
| `./mvnw test -Dtest='TimeRoundingTest#rejects*'` | Methods whose name starts with `rejects` |
| `./mvnw test -Dtest='*BillingTest'` | All classes whose name ends in `BillingTest` |
| `./mvnw verify` | Everything CI runs: Spotless check, compile, tests, (after step 10) coverage report |

In zsh and bash, put the pattern in quotes, otherwise the shell tries to expand `*` itself. In IntelliJ
you can run a single row of a parameterized test from the run window.

Reports: Surefire writes one file per test class to `target/surefire-reports/`. When CI fails, the
`.txt` file of the failing class has the full message.

---

## Step 9 — One TDD cycle: `Money.sum()`

**Why:** in lesson 13 an invoice subtotal is the sum of its lines. A `Money.sum(...)` that takes any
number of amounts is clearer than a loop with `add()` in the invoice code, and it belongs in `Money`. We
add it the TDD way: test first (README, section 12).

**Red.** Decide how the method looks *from the caller's side* and write that down as tests. Add a nested
group to `MoneyTest`:

```java
// src/test/java/ee/ta25/billable/common/MoneyTest.java — add inside class MoneyTest
  @Nested
  class Summing {

    @Test
    void the_sum_of_no_amounts_is_zero() {
      assertThat(Money.sum()).isEqualTo(Money.zero());
    }

    @Test
    void the_sum_of_one_amount_is_that_amount() {
      assertThat(Money.sum(Money.fromCents(1250))).isEqualTo(Money.fromCents(1250));
    }

    @Test
    void sums_several_amounts() {
      var total = Money.sum(Money.fromCents(1250), Money.fromCents(399), Money.fromCents(-100));

      assertThat(total).isEqualTo(Money.fromCents(1549));
    }
  }
```

The three tests follow a classic rule for anything that takes a list: **zero, one, many**. "Zero
items" is the boundary where loops most often go wrong — what is the start value?

Run `./mvnw test -Dtest='MoneyTest*'`:

```text
[ERROR] COMPILATION ERROR :
[ERROR] .../MoneyTest.java:[..] cannot find symbol
  symbol:   method sum()
  location: class ee.ta25.billable.common.Money
```

In Java, "red" for a method that does not exist yet is a **compile error**. That still counts: the tests
describe behaviour the code does not have.

**Green.** The simplest implementation. Add to the `Money` record, below the factory methods:

```java
// src/main/java/ee/ta25/billable/common/Money.java — add inside the record
  public static Money sum(Money... amounts) {
    var total = zero();
    for (var amount : amounts) {
      total = total.add(amount);
    }
    return total;
  }
```

Run again → all green.

**What happens:** `Money... amounts` is a **varargs** parameter: the method accepts any number of `Money`
arguments, and inside it `amounts` is an array. With no arguments the array is empty, the loop does not
run and the result is zero. A caller with a list writes
`Money.sum(lines.stream().map(InvoiceLine::amount).toArray(Money[]::new))` (lesson 13). Because each step
uses `add()`, the overflow check from lesson 10 still protects the sum. The Laravel track has the same
signature (`Money ...$amounts`).

**Refactor.** With the tests green, look at the code. Java streams have an operation for "combine a list
into one value" — `reduce`, with a start value and a combining function:

```java
// src/main/java/ee/ta25/billable/common/Money.java — sum() after refactoring
  public static Money sum(Money... amounts) {
    return Arrays.stream(amounts).reduce(zero(), Money::add);
  }
```

(Add `import java.util.Arrays;`.) `Money::add` is a **method reference**: short for
`(total, amount) -> total.add(amount)`. Run the tests again → still green. That is the point of the refactor step: you changed *how* the method works, and the
tests prove that *what* it does stayed the same. (If the loop is easier for you to read, keep the loop —
refactoring means "better", not "shorter".)

**Commit:** `feat(money): add Money.sum()` (tests and code in one commit)

---

## Step 10 — Code coverage with JaCoCo

**Why:** coverage shows which lines and branches no test executes (README, section 11). **JaCoCo** is the
standard tool for Java. It works as a Java *agent*: while the tests run, it records which bytecode
instructions were executed, and afterwards it writes a report.

Add the plugin to the `<build><plugins>` section of `pom.xml`, next to Spotless:

```xml
<!-- pom.xml — inside <build><plugins> -->
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <!-- Use the latest version from Maven Central. Java 25 needs 0.8.14 or newer. -->
  <version>0.8.15</version>
  <executions>
    <execution>
      <id>prepare-agent</id>
      <goals>
        <goal>prepare-agent</goal>
      </goals>
    </execution>
    <execution>
      <id>report</id>
      <phase>verify</phase>
      <goals>
        <goal>report</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

**What happens:**

1. `prepare-agent` runs before the tests. It sets a Maven property called `argLine` that Surefire uses
   to start the test JVM with the JaCoCo agent attached.
2. The agent writes the raw data to `target/jacoco.exec` while the tests run.
3. `report` (bound to the `verify` phase) turns that data into HTML in `target/site/jacoco/`.
4. **The version matters.** JaCoCo reads your compiled classes. A JaCoCo release older than your JDK
   cannot read the class files and fails with `Unsupported class file major version 69` (69 = Java 25).
   Check Maven Central for the newest release when you add it.

Run:

```bash
./mvnw verify
```

**Check it works:** open `target/site/jacoco/index.html` in the browser. You see one row per package with
**instruction** and **branch** coverage. Click `ee.ta25.billable.common` → `Money` → a method: green lines
are covered, red lines are not, and yellow diamonds mark branches where only one side (true *or* false)
was executed.

Expect a low total: controllers, repositories and services are not unit tested (they get feature and
integration tests in lessons 12 and 14). Look at `common` and `billing` — those are today's target.

If `contextLoads` fails because Docker Desktop is not running, `verify` stops before the report. Start
Docker Desktop and run it again.

CI already runs `./mvnw verify` (lesson 02), so it now also creates the report. The tests themselves
needed no CI change at all: `verify` includes the `test` phase.

**Commit:** `build: add JaCoCo coverage report`

---

## Step 11 — A deliberately bad test

**Why:** you learn as much from a bad test as from a good one. **Do not add this to your project** —
read it, find the problems, then compare with the table.

```java
// A bad test — for reading only, do not type it
@SpringBootTest
class BillingTests {

  @Autowired ProjectRepository projectRepository;

  @Test
  void test() {
    long minutes = new Random().nextLong(1, 600);
    Money expected = Money.fromCents((minutes * 6000 + 30) / 60);
    Project project = projectRepository.findAll().getFirst();

    Money result = new HourlyBilling().calculate(project, minutes, Money.fromCents(0));

    assertThat(result.greaterThanOrEqual(Money.fromCents(0))).isTrue();
  }
}
```

| Problem | Consequence |
|---|---|
| `@SpringBootTest` for a pure calculation | Starts the whole application and needs the database: seconds instead of a millisecond |
| Name `test()` | When it fails, nobody knows what broke |
| `new Random()` | A failure cannot be reproduced |
| Expected value uses the production formula | If the formula is wrong, the test is wrong the same way and stays green |
| `findAll().getFirst()` | Depends on whatever data is in your local database; different in CI |
| `expected` is never used | The only check is "not negative" — a method that always returns zero passes |

Coverage would show `HourlyBilling` at 100 % with this test. That is the lesson: coverage shows what was
**executed**, not what was **checked**. The good version is `bills_ninety_minutes_at_sixty_euros_per_hour`
from step 6: fixed input, a hand-calculated expected value, an in-memory project, an exact assertion, no
Spring.

---

## Troubleshooting

| Error or symptom | Cause | Fix |
|---|---|---|
| `package org.junit does not exist` / `@Before` not found | Code from a JUnit 4 tutorial | Use `org.junit.jupiter.api.Test`, `@BeforeEach`, `@ExtendWith` |
| Test class compiles but `Tests run: 0` | Class name does not end in `Test`/`Tests`, or `@Test` imported from `org.junit` (JUnit 4) | Rename the class; import `org.junit.jupiter.api.Test` |
| `@Nested` tests do not run with `-Dtest=MoneyTest` | The pattern does not match `MoneyTest$Arithmetic` | Use `-Dtest='MoneyTest*'` |
| `zsh: no matches found: -Dtest=MoneyTest*` | The shell expands `*` | Put the value in quotes |
| `ParameterResolutionException: No ParameterResolver registered for parameter [long arg0]` | `@Test` on a method with parameters | Use `@ParameterizedTest` plus a source annotation |
| `ArgumentConversionException` in a `@CsvSource` row | A value cannot be converted (e.g. `12.50 €` into `long`), or a comma inside a value | Check column order and types; use `delimiter = '\|'` or quote the value with `'...'` |
| `cannot find symbol: method setBudgetCapCents` in `ProjectTestData` | Your `Project` has other constructor/setter names | Adapt `build()` — only there |
| `Expecting code to raise a throwable.` | The code did not throw for invalid input | Decide the rule and add the check in production code |
| `Unsupported class file major version 69` from JaCoCo | JaCoCo too old for Java 25 | Update `jacoco-maven-plugin` to the latest version |
| `The following files had format violations` during `verify` | New test files not formatted | `./mvnw spotless:apply` |
| `contextLoads` fails: `Could not find a valid Docker environment` | Docker Desktop not running (Testcontainers needs it) | Start Docker Desktop |

---

## Recap

- Unit tests live in `src/test/java` in the package of the class under test, and have **no Spring
  annotations** — JUnit creates them with `new`.
- New files: `common/MoneyTest.java`, `common/TimeRoundingTest.java`, `common/DueDateCalculatorTest.java`,
  `billing/HourlyBillingTest.java`, `CappedHourlyBillingTest.java`, `FixedPriceBillingTest.java`, and the
  builder `project/ProjectTestData.java`.
- `@Nested` groups related tests; `@ParameterizedTest` + `@CsvSource` / `@ValueSource` turns a boundary
  table into one test with named rows.
- `assertThatThrownBy(() -> ...)` checks the error cases.
- Entities are plain objects: `new Project(...)` in a test touches no database. The test data builder
  keeps the constructor call in one place.
- `Money.sum()` was added test-first (zero, one, many).
- JaCoCo writes `target/site/jacoco/index.html` during `./mvnw verify`; CI runs the tests with no change.

---

## Independent work — solutions

> **The tasks are not here.** The task descriptions (Basic / Intermediate / Advanced, with acceptance
> criteria) are in this lesson's [README.md → Independent work (~2 h)](README.md#independent-work-2-h).
> Read them there first and build the features in your project. This section only contains the
> solutions — try each task yourself, then open a solution to compare or when you are stuck.

<details>
<summary><strong>Task 1 (Basic)</strong> — 15+ meaningful unit tests, every billing type at its boundaries</summary>

**Capped:** with a 1000 € cap and 600 minutes at 60 €/h (= 600 €), the remaining budget is exactly 600 €
when 400 € is already billed. Add to `CappedHourlyBillingTest`:

```java
// src/test/java/ee/ta25/billable/billing/CappedHourlyBillingTest.java — add inside the class,
// plus imports org.junit.jupiter.params.ParameterizedTest and ...provider.CsvSource
  @ParameterizedTest(name = "already billed {0} cents -> bills {1} cents")
  @CsvSource(
      textBlock =
          """
          # hourly amount for 600 minutes is always 60000 cents (600.00 €)
          # remaining 600.01 € is more than hourly
          39999, 60000
          # remaining exactly 600.00 € equals hourly
          40000, 60000
          # remaining 599.99 € is less than hourly
          40001, 59999
          # one cent left
          99999, 1
          """)
  void bills_the_smaller_of_hourly_amount_and_remaining_budget(
      long alreadyBilledCents, long expectedCents) {
    assertThat(strategy.calculate(project, 600, Money.fromCents(alreadyBilledCents)))
        .isEqualTo(Money.fromCents(expectedCents));
  }
```

**Fixed price:**

```java
// src/test/java/ee/ta25/billable/billing/FixedPriceBillingTest.java — add inside the class,
// plus imports for ParameterizedTest and CsvSource
  @ParameterizedTest(name = "already billed {0} cents -> bills {1} cents")
  @CsvSource({"249999, 1", "250000, 0", "300000, 0"})
  void bills_what_is_left_of_the_price_but_never_less_than_zero(
      long alreadyBilledCents, long expectedCents) {
    assertThat(strategy.calculate(project, 600, Money.fromCents(alreadyBilledCents)))
        .isEqualTo(Money.fromCents(expectedCents));
  }

  @Test
  void the_number_of_minutes_does_not_change_a_fixed_price() {
    assertThat(strategy.calculate(project, 0, Money.fromCents(0)))
        .isEqualTo(Money.fromCents(250_000))
        .isEqualTo(strategy.calculate(project, 6000, Money.fromCents(0)));
  }
```

**Hourly:**

```java
// src/test/java/ee/ta25/billable/billing/HourlyBillingTest.java — add inside the class
  @Test
  void rounds_just_below_half_a_cent_down() {
    // 90.29 € per hour for 1 minute = 150.483 cents
    var project = ProjectTestData.hourly(9029).build();

    assertThat(strategy.calculate(project, 1, Money.fromCents(0))).isEqualTo(Money.fromCents(150));
  }

  @Test
  void ignores_what_was_already_billed() {
    var project = ProjectTestData.hourly(6000).build();

    assertThat(strategy.calculate(project, 60, Money.fromCents(100_000)))
        .isEqualTo(Money.fromCents(6000));
  }
```

**Counting.** List behaviours, not rows:

| Class | Behaviours tested |
|---|---|
| `Money` | add, subtract, multiply, parse valid, parse invalid, half-up percentage (incl. negative), isZero, isNegative, gte equal, gte smaller, format, sum ×3 |
| `TimeRounding` | round up at 1/6/15, invalid input |
| `HourlyBilling` | normal, zero, negative minutes rejected, half cent up, just below half, ignores already billed |
| `CappedHourlyBilling` | far below cap, remaining vs hourly (table), cap reached, over cap |
| `FixedPriceBilling` | full, partial, one cent left / fully billed / over-billed, minutes irrelevant |
| `DueDateCalculator` | weekend rule (table) |

**Commit:** `test(billing): boundary tables for capped and fixed-price strategies`

</details>

<details>
<summary><strong>Task 2 (Basic)</strong> — extract <code>BudgetMonitor.reachedThreshold()</code> and test it with a boundary table</summary>

**1. The pure method.** Your `BudgetMonitor` already compares exactly (lessons 09–10), but the
comparison is buried in `check()` next to the repositories and the notifier. Add a static method:

```java
// src/main/java/ee/ta25/billable/budget/BudgetMonitor.java — add inside the class, below check()
  /** True when {@code value} is at least 80 % of {@code cap} (R9). Exact, no rounding. */
  public static boolean reachedThreshold(Money value, Money cap) {
    return value.multiply(100).greaterThanOrEqual(cap.multiply(WARNING_THRESHOLD_PERCENT));
  }
```

In `check()`, replace your comparison (for example `value.multiply(100)...` or
`Math.multiplyExact(valueCents, 100) < Math.multiplyExact(capCents, WARNING_THRESHOLD_PERCENT)`) with:

```java
// src/main/java/ee/ta25/billable/budget/BudgetMonitor.java — inside check()
    if (!reachedThreshold(value, cap)) {
      return;
    }
```

(`value` and `cap` are the two `Money` values `check()` already computes.)

**Why exact comparison?** (This is the trap the task hint points at.) `cap.percentageBp(8000)` rounds.
For a 9,99 € cap it gives 799.2 → 7,99 €, and a value of 7,99 € (only 79.98 %) would trigger the warning.
`value × 100 >= cap × 80` never divides: `79 900 < 79 920` → no warning. Correct. The test has a row for
exactly this case, so nobody can "simplify" the method later without a red test.

**Why static?** It uses nothing from the object — no repository, no notifier. The test calls it without
creating a `BudgetMonitor` or any mocks.

**2. The test:**

```java
// src/test/java/ee/ta25/billable/budget/BudgetMonitorThresholdTest.java
package ee.ta25.billable.budget;

import static org.assertj.core.api.Assertions.assertThat;

import ee.ta25.billable.common.Money;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

class BudgetMonitorThresholdTest {

  @ParameterizedTest(name = "value {0} of cap {1} -> warn: {2}")
  @CsvSource(
      textBlock =
          """
          # nothing logged (0 %)
          0, 10000, false
          # one cent below (79.99 %)
          7999, 10000, false
          # exactly 80 %
          8000, 10000, true
          # one cent above (80.01 %)
          8001, 10000, true
          # the whole cap (100 %)
          10000, 10000, true
          # over the cap (120 %)
          12000, 10000, true
          # odd cap 9.99 €: 7.99 € is only 79.98 %
          799, 999, false
          # odd cap 9.99 €: 8.00 € is 80.08 %
          800, 999, true
          """)
  void decides_whether_80_percent_of_the_cap_is_reached(
      long valueCents, long capCents, boolean expected) {
    assertThat(BudgetMonitor.reachedThreshold(Money.fromCents(valueCents), Money.fromCents(capCents)))
        .isEqualTo(expected);
  }
}
```

**3. Check.** `./mvnw test -Dtest=BudgetMonitorThresholdTest` → 8 rows green. Log time on a capped project
in the browser until it crosses 80 % and check Mailpit (`http://localhost:8025`): the warning still
arrives, once.

**Commit:** `refactor(budget): extract reachedThreshold() and cover it with a boundary table`

</details>

<details>
<summary><strong>Task 3 (Intermediate)</strong> — coverage report and <code>docs/testing.md</code></summary>

Generate the report with `./mvnw verify` and open `target/site/jacoco/index.html`. `target/` is already
in the `.gitignore` that Initializr created — check it, generated files do not belong in Git.

Optional: let CI keep the report so you can download it from the Actions run. Add after the `verify` step
in `.github/workflows/ci.yml`:

```yaml
# .github/workflows/ci.yml — add as the last step of the build job
      - name: Upload coverage report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: jacoco-report
          path: target/site/jacoco/
```

Example `docs/testing.md` (use your own numbers and reasons):

```markdown
<!-- docs/testing.md -->
# Testing

## Running the tests

- All tests: `./mvnw test` (Docker Desktop must run: `contextLoads` starts a PostgreSQL container)
- One class: `./mvnw test -Dtest=TimeRoundingTest`
- Coverage report: `./mvnw verify`, then open `target/site/jacoco/index.html`

## Coverage of the domain code

| Package | Instructions | Branches |
|---|---|---|
| `ee.ta25.billable.common` | 96 % | 92 % |
| `ee.ta25.billable.billing` | 88 % | 100 % |

## What is not unit tested, and why

- **`BillingStrategyResolver`.** It builds a map from the injected strategies (keyed by each
  strategy's `type()`), and `forProject()` looks up the project's billing type. A unit test would
  repeat the map construction. What matters is that Spring injects all three strategies, which only
  an integration test (lesson 14) can show.
- **`BudgetMonitor.check()`.** It loads data through repositories and calls the notifier. The
  decision (80 %) is extracted into `reachedThreshold()` and unit tested; the rest is tested with
  Mockito in lesson 12.
- **Controllers and forms.** No calculations; their job is binding, validation and choosing a view.
  They get a `@WebMvcTest` in lesson 12.
- **`Money.format()` with a locale.** `format()` itself is tested (`Formatting`); the optional
  `format(Locale)` from the lesson 10 advanced task depends on JDK locale data and is not tested yet.
```

Every "not tested" item has a **reason** and says **where else** it is covered, or admits that it is not.

**Commit:** `docs(testing): coverage report and what is left untested`

</details>

<details>
<summary><strong>Task 4 (Advanced)</strong> — mutation testing with PIT</summary>

**PIT** (pitest) changes your bytecode in small ways (**mutants**: `>=` → `>`, `+` → `-`, return values
replaced) and runs the tests against each one. A mutant that makes a test fail is **killed**. A
**surviving** mutant is a change no test notices.

Add the plugin to `<build><plugins>`. Look up the latest versions of `pitest-maven` and
`pitest-junit5-plugin` on Maven Central (group `org.pitest`) and put them in the two `<version>`
elements:

```xml
<!-- pom.xml — inside <build><plugins> -->
<plugin>
  <groupId>org.pitest</groupId>
  <artifactId>pitest-maven</artifactId>
  <version><!-- latest pitest-maven from Maven Central --></version>
  <dependencies>
    <dependency>
      <groupId>org.pitest</groupId>
      <artifactId>pitest-junit5-plugin</artifactId>
      <version><!-- latest pitest-junit5-plugin from Maven Central --></version>
    </dependency>
  </dependencies>
  <configuration>
    <targetClasses>
      <param>ee.ta25.billable.common.*</param>
      <param>ee.ta25.billable.billing.*</param>
      <param>ee.ta25.billable.budget.BudgetMonitor</param>
    </targetClasses>
    <targetTests>
      <param>ee.ta25.billable.common.*</param>
      <param>ee.ta25.billable.billing.*</param>
      <param>ee.ta25.billable.budget.BudgetMonitorThresholdTest</param>
    </targetTests>
  </configuration>
</plugin>
```

**What happens:**

1. PIT is not bound to any phase, so a normal `./mvnw verify` does not run it (it is slow). You start it
   by hand.
2. `pitest-junit5-plugin` lets PIT run JUnit Jupiter tests. Without it, PIT finds no tests. If PIT reports
   "no tests found" or a JUnit Platform version conflict, check the plugin's release notes for support of
   the JUnit version Spring Boot 4 uses (JUnit 6).
3. `targetClasses` limits mutation to the pure code. Mutating controllers would only produce noise.
   `targetTests` keeps `contextLoads` (which needs the database) out. `BudgetMonitor` is in the list
   for `reachedThreshold()` from Task 2; the mutants in its `check()` show as **no coverage**, because
   only lesson 12 tests `check()`.

Run:

```bash
./mvnw test-compile org.pitest:pitest-maven:mutationCoverage
```

**Check it works:** open `target/pit-reports/index.html` (older versions add a timestamp folder). You see
line coverage and **mutation coverage** per class; click a class to see each mutant, killed (green) or
survived (red).

**An example you are likely to see.** PIT's *conditionals boundary* mutator changes `>=` into `>`. In
`Money.greaterThanOrEqual()` and in `BudgetMonitor.reachedThreshold()` that mutant **survives** if no test
uses two equal values — "exactly 80 %" or "500 >= 500". The rows `an_equal_amount_counts_as_greater_or_equal`
and `8000, 10000, true` kill it. The *math* mutator changes `divisor / 2` into `divisor * 2` in
`Money.divideHalfUp()`: ÷ 60 then adds 120 instead of 30, 90 minutes at 6000 gives 9002 instead of 9000,
so the 90-minute test kills it (and so do the percentage rows, because VAT uses the same function). Every
surviving mutant points at a boundary that no test sits on.

**Equivalent mutants** change the code without changing behaviour (for example a condition on a value
that an earlier check already excludes). No test can kill them; note why they are harmless instead of
writing a meaningless test.

Add to `docs/testing.md`: the mutation score, one surviving mutant, and what you did about it.

**Commit:** `test: add PIT mutation testing for common, billing and the budget threshold`

</details>
