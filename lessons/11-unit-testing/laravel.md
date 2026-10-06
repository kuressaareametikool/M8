# 11 — Unit testing · Laravel

[← Concepts](README.md) · Starting point: end of lesson 10 · Estimated time in class: 2 h

---

## What we build today

- A clean `tests/Unit` folder whose tests extend `PHPUnit\Framework\TestCase` — no framework boot, no
  database.
- `tests/Unit/Support/MoneyTest.php` — arithmetic, parsing euro strings (valid and invalid), half-up
  rounding with basis points, comparison and formatting.
- `tests/Unit/Support/TimeRoundingTest.php` — a data-driven boundary table with `#[DataProvider]`.
- `tests/Unit/Billing/HourlyBillingTest.php`, `CappedHourlyBillingTest.php`, `FixedPriceBillingTest.php`
  — the three strategies, with `Project` objects built in memory.
- `tests/Unit/Support/DueDateCalculatorTest.php` — the weekend rule (R12).
- One TDD cycle that adds `Money::sum()`.
- A deliberately bad test, and its fixed version.
- A `tests` job in the GitHub Actions workflow.

**Final result:** `php artisan test --testsuite=Unit` runs about 65 test cases in well under a second and
prints a green list of sentences like "✓ rounds sixteen minutes up to thirty". The same tests run in CI
on every push.

### The lesson 10 code we test

Tests are written against the **public behaviour** of a class. This is the behaviour this guide expects
from lesson 10. If your method names differ slightly, change the tests, not the lesson 10 code — unless
a test finds a real bug.

| Class | Behaviour the tests rely on |
|---|---|
| `App\Support\Money` | `Money::zero()`, `Money::fromCents(int)`, `Money::fromEuros(string)` (accepts `"85.50"`, `"85,5"`, `"1200"`, `"-3.10"`, ignores surrounding spaces; throws `InvalidArgumentException` for more than two decimals or anything else), `add()`, `subtract()`, `multiply(int)`, `percentageBp(int)` (half up, away from zero), `min()`, `max()`, `isZero()`, `isNegative()`, `greaterThanOrEqual()`, `format()` (Estonian style `"1 234,50 €"`). The cents are in `public readonly int $cents`. |
| `App\Support\TimeRounding` | `TimeRounding::roundUp(int $minutes, int $increment): int`; throws `InvalidArgumentException` for negative minutes or an increment other than 1, 6 or 15 |
| `App\Billing\HourlyBilling` | also `HourlyBilling::forMinutes(Money $rate, int $minutes)`, which throws `InvalidArgumentException` for negative minutes |
| `App\Billing\*` | `calculate(Project $project, int $billableMinutes, Money $alreadyBilled): Money` on `HourlyBilling`, `CappedHourlyBilling`, `FixedPriceBilling`. `HourlyBilling` and `FixedPriceBilling` have no constructor arguments; `CappedHourlyBilling` receives an `HourlyBilling` in its constructor (lesson 08) |
| `App\Support\DueDateCalculator` | `(new DueDateCalculator)->dueDate(CarbonImmutable $issuedOn): CarbonImmutable` |

---

## Step 1 — Look at the test setup Laravel gave you

**Why:** before writing tests, know where they live, how they are found and what the two folders mean.
The Laravel installer created all of this when you chose PHPUnit in lesson 01.

Open `phpunit.xml` in the project root. The important parts:

```xml
<!-- phpunit.xml (excerpt — already in your project; see the note below about the two DB lines) -->
<testsuites>
    <testsuite name="Unit">
        <directory>tests/Unit</directory>
    </testsuite>
    <testsuite name="Feature">
        <directory>tests/Feature</directory>
    </testsuite>
</testsuites>
<source>
    <include>
        <directory>app</directory>
    </include>
</source>
<php>
    <env name="APP_ENV" value="testing"/>
    <env name="DB_CONNECTION" value="sqlite"/>
    <env name="DB_DATABASE" value=":memory:"/>
    <!-- ... more env lines ... -->
</php>
```

**What happens:**

1. `<testsuites>` defines two groups. PHPUnit finds every file ending in `Test.php` in those folders.
2. **Unit** tests in `tests/Unit` extend `PHPUnit\Framework\TestCase`. That is plain PHPUnit: no Laravel
   application, no container, no facades, no database. They start in about a millisecond.
3. **Feature** tests in `tests/Feature` extend `Tests\TestCase`. That class boots the whole Laravel
   application before each test, so you can make HTTP requests and use the database. That costs
   10–100 ms per test. We use them in lessons 12 and 14.
4. `<source>` tells PHPUnit which code belongs to *your* application. It is used for the coverage report
   (step 8).
5. The `<env>` lines override `.env` while tests run. Feature tests use an SQLite database in memory — not
   your PostgreSQL. Unit tests do not use a database at all.

**Check the two database lines now.** In a new Laravel project they are **commented out**
(`<!-- <env name="DB_CONNECTION" value="sqlite"/> -->`). If you did the lesson 06 advanced task, you have
already removed the `<!--` and `-->`. If not, do it now, so the file contains exactly the two lines shown
above. Without them, feature tests with `RefreshDatabase` (lessons 12–14) would run against your
development PostgreSQL database and **delete your data**. Lesson 14 discusses the SQLite trade-off.

Run the tests that already exist:

```bash
php artisan test
```

**Check it works:**

```text
   PASS  Tests\Unit\ExampleTest
  ✓ that true is true                                                    0.01s

   PASS  Tests\Feature\ExampleTest
  ✓ the application returns a successful response                        0.10s

  Tests:    2 passed (2 assertions)
  Duration: 0.21s
```

The unit example asserts that `true` is `true` — it tests nothing. Delete it:

```bash
rm tests/Unit/ExampleTest.php
```

Keep `tests/Feature/ExampleTest.php` for now: it checks that the home page answers with status 200, which
is a real (if small) feature test. Since lesson 03 the home page counts clients and projects, so this test
needs the tables. With SQLite in memory the database starts empty, and the test fails with `no such table:
clients`. Add the `RefreshDatabase` trait, which runs your migrations for the test:

```php
<?php

// tests/Feature/ExampleTest.php

namespace Tests\Feature;

use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    use RefreshDatabase;

    public function test_the_application_returns_a_successful_response(): void
    {
        $response = $this->get('/');

        $response->assertStatus(200);
    }
}
```

> **Version note.** Laravel 12 installs PHPUnit 11; newer Laravel versions may install PHPUnit 12. Old
> tutorials put test metadata in doc comments (`/** @test */`, `/** @dataProvider cases */`). PHPUnit 11
> warns that this is deprecated, and **PHPUnit 12 ignores it completely** — the test or data provider
> silently stops working. Use PHP attributes (`#[DataProvider('cases')]`, `#[Test]`) as in this guide.

---

## Step 2 — The first test: adding money

**Why:** we start with the smallest possible test to see the whole cycle: write, run, fail, pass.

Create the folder `tests/Unit/Support` and the file:

```php
<?php

// tests/Unit/Support/MoneyTest.php

namespace Tests\Unit\Support;

use App\Support\Money;
use PHPUnit\Framework\TestCase;

class MoneyTest extends TestCase
{
    public function test_adds_two_amounts(): void
    {
        // Arrange
        $price = Money::fromCents(1250);
        $extra = Money::fromCents(399);

        // Act
        $total = $price->add($extra);

        // Assert
        $this->assertEquals(Money::fromCents(1649), $total);
    }
}
```

**What happens:**

1. The namespace `Tests\Unit\Support` mirrors the folder, like `App\Support` mirrors `app/Support`.
   Composer's `autoload-dev` section maps `Tests\` to `tests/`.
2. `extends TestCase` — the **PHPUnit** one (`PHPUnit\Framework\TestCase`), not `Tests\TestCase`. That
   is the whole difference between a unit test and a feature test in Laravel. Check the `use` line
   whenever something is slow.
3. PHPUnit runs every public method whose name starts with `test`. Laravel's convention is
   `snake_case` names; Pint's `laravel` preset keeps them that way. The output turns `test_adds_two_amounts`
   into "adds two amounts".
4. **Arrange** builds two amounts. **Act** calls the one method under test. **Assert** compares.
5. `assertEquals(expected, actual)` on two objects compares their class and all properties. So it
   checks that the result is a `Money` with `$cents === 1649`. If it fails, PHPUnit prints both objects.

Run only this file:

```bash
php artisan test --filter=MoneyTest
```

**Check it works:** `✓ adds two amounts` and `Tests: 1 passed (1 assertions)`.

Now **make it fail on purpose** — this proves the test can see a bug. In `app/Support/Money.php`, change
`add()` to subtract (for example `$this->cents - $other->cents`) and run the test again:

```text
   FAIL  Tests\Unit\Support\MoneyTest
  ⨯ adds two amounts

  Failed asserting that two objects are equal.
  --- Expected
  +++ Actual
  @@ @@
   App\Support\Money Object (
  -    'cents' => 1649
  +    'cents' => 851
   )
```

The `-` line is what you expected, the `+` line is what the code returned. Undo the change and run again —
green. Every time you write a new kind of test, break the code once and watch it go red.

---

## Step 3 — The rest of `MoneyTest`

**Why:** `Money` is used by every invoice. We test each public behaviour, and for parsing and rounding we
use **boundary tables** (README, section 7).

Replace the whole file with this version:

```php
<?php

// tests/Unit/Support/MoneyTest.php

namespace Tests\Unit\Support;

use App\Support\Money;
use InvalidArgumentException;
use PHPUnit\Framework\Attributes\DataProvider;
use PHPUnit\Framework\TestCase;

class MoneyTest extends TestCase
{
    public function test_adds_two_amounts(): void
    {
        $total = Money::fromCents(1250)->add(Money::fromCents(399));

        $this->assertEquals(Money::fromCents(1649), $total);
    }

    public function test_subtracts_an_amount(): void
    {
        $rest = Money::fromCents(1000)->subtract(Money::fromCents(250));

        $this->assertEquals(Money::fromCents(750), $rest);
    }

    public function test_multiplies_by_a_whole_number(): void
    {
        $this->assertEquals(Money::fromCents(3750), Money::fromCents(1250)->multiply(3));
    }

    public function test_does_not_change_the_original_amount(): void
    {
        $price = Money::fromCents(1250);

        $price->add(Money::fromCents(100));

        $this->assertEquals(Money::fromCents(1250), $price);
    }

    #[DataProvider('validEuroStrings')]
    public function test_parses_a_euro_string(string $input, int $expectedCents): void
    {
        $this->assertEquals(Money::fromCents($expectedCents), Money::fromEuros($input));
    }

    public static function validEuroStrings(): array
    {
        return [
            'whole euros' => ['12', 1200],
            'one decimal' => ['12.5', 1250],
            'two decimals' => ['12.50', 1250],
            'comma as decimal separator' => ['12,50', 1250],
            'smallest amount' => ['0.01', 1],
            'zero' => ['0', 0],
            'four digits' => ['1234.56', 123456],
            'negative amount' => ['-3.10', -310],
            'surrounding spaces are ignored' => [' 12.50 ', 1250],
        ];
    }

    #[DataProvider('invalidEuroStrings')]
    public function test_rejects_an_invalid_euro_string(string $input): void
    {
        $this->expectException(InvalidArgumentException::class);

        Money::fromEuros($input);
    }

    public static function invalidEuroStrings(): array
    {
        return [
            'empty' => [''],
            'not a number' => ['abc'],
            'three decimals' => ['12.345'],
            'separator without decimals' => ['12.'],
            'thousands separator' => ['1.234,56'],
        ];
    }

    #[DataProvider('percentageCases')]
    public function test_takes_a_percentage_rounded_half_up(int $cents, int $basisPoints, int $expectedCents): void
    {
        $this->assertEquals(
            Money::fromCents($expectedCents),
            Money::fromCents($cents)->percentageBp($basisPoints),
        );
    }

    public static function percentageCases(): array
    {
        return [
            '24 % of 12.50 is exactly 3.00' => [1250, 2400, 300],
            '24 % of 10.21 is 2.4504, rounds down' => [1021, 2400, 245],
            '24 % of 10.23 is 2.4552, rounds up' => [1023, 2400, 246],
            '50 % of 1 cent is exactly half, rounds up' => [1, 5000, 1],
            '50 % of 3 cents is 1.5, rounds up' => [3, 5000, 2],
            '49.99 % of 1 cent is just below half' => [1, 4999, 0],
            '24 % of nothing is nothing' => [0, 2400, 0],
            'negative half rounds away from zero' => [-1, 5000, -1],
            'negative 1.5 rounds away from zero' => [-3, 5000, -2],
        ];
    }

    public function test_knows_when_it_is_zero(): void
    {
        $this->assertTrue(Money::fromCents(0)->isZero());
        $this->assertFalse(Money::fromCents(1)->isZero());
    }

    public function test_a_result_below_zero_is_negative(): void
    {
        $this->assertTrue(Money::fromCents(100)->subtract(Money::fromCents(250))->isNegative());
        $this->assertFalse(Money::fromCents(0)->isNegative());
    }

    public function test_an_equal_amount_counts_as_greater_or_equal(): void
    {
        $this->assertTrue(Money::fromCents(500)->greaterThanOrEqual(Money::fromCents(500)));
    }

    public function test_a_smaller_amount_is_not_greater_or_equal(): void
    {
        $this->assertFalse(Money::fromCents(499)->greaterThanOrEqual(Money::fromCents(500)));
    }

    #[DataProvider('formatCases')]
    public function test_formats_as_euros(int $cents, string $expected): void
    {
        $this->assertSame($expected, Money::fromCents($cents)->format());
    }

    public static function formatCases(): array
    {
        return [
            'normal amount' => [1250, '12,50 €'],
            'less than ten cents keeps the leading zero' => [5, '0,05 €'],
            'zero' => [0, '0,00 €'],
            'thousands are separated by a space' => [123456, '1 234,56 €'],
            'negative amount' => [-5, '-0,05 €'],
        ];
    }
}
```

**What happens:**

1. `test_does_not_change_the_original_amount` checks that `Money` is **immutable**: `add()` returns a new
   object and leaves the old one alone. If somebody later "optimises" `add()` to change `$this`, invoices
   that reuse an amount would silently change — this test catches it.
2. `#[DataProvider('validEuroStrings')]` is a PHP **attribute** (the `#[...]` syntax). It tells PHPUnit:
   "call `validEuroStrings()`, and run this test once for every row". Each row is an array of arguments
   in the order of the test method's parameters.
3. The provider method must be `public static`. PHPUnit calls it *before* any test object exists, so it
   cannot use `$this`. (PHPUnit 10 made this a hard rule.)
4. The **array keys** (`'whole euros'`, …) name each row. When a row fails, the output says
   `with data set "three decimals"` instead of `with data set #2`.
5. `expectException(InvalidArgumentException::class)` must come **before** the call. It tells PHPUnit
   "this test passes only if this exception is thrown from here on". If `fromEuros()` returns normally,
   the test fails with `Failed asserting that exception of type "InvalidArgumentException" is thrown`.
6. The percentage table has the **boundaries of half-up rounding**: exactly half (1 cent × 50 %), just
   below half (49.99 %), and normal VAT cases that round down and up. The two negative rows check the
   lesson 10 decision that "half up" means *away from zero* (−0.5 → −1). Look at the row names — they
   are the documentation of R3.
7. An interesting discovery while choosing rows: with 24 % VAT you can **never** hit exactly half a
   cent. That would need `cents × 2400` to end in `5000`, so `cents × 24` would have to end in `50`. But
   `cents × 24` is always a multiple of 4, and a number ending in `50` never is. So to test "exactly
   half rounds up" we use 50 % — the formula is the same, only the rate differs.
8. `assertSame()` for the format string: for scalar values `assertSame` also checks the **type**, so
   `'12,50 €'` must be a string, not something that merely converts to it. `format()` uses a normal
   space as the thousands separator; if you switched to the `NumberFormatter` version from the lesson 10
   advanced task, that is a non-breaking space and the expected string must contain `\u{00A0}`.
9. `test_knows_when_it_is_zero` has two assertions. That is fine here: together they describe one
   behaviour ("isZero is true for zero and only for zero"). If the two assertions described different
   behaviours, we would split them.

If your `format()` produces a different style, change the expected strings to your format — the test
documents **your** decision.

**Check it works:**

```bash
php artisan test --filter=MoneyTest
```

```text
   PASS  Tests\Unit\Support\MoneyTest
  ✓ adds two amounts
  ✓ subtracts an amount
  ...
  ✓ parses a euro string with data set "whole euros"
  ...
  ✓ rejects an invalid euro string with data set "thousands separator"
  ...
  Tests:    36 passed
```

If a `fromEuros` row fails, first decide who is right. Maybe your lesson 10 code does not accept a comma,
or accepts a thousands separator. Either is a design decision — make it on purpose, and move the row to
the table that matches your decision.

**Commit:** `test(money): cover arithmetic, parsing, rounding and formatting`

---

## Step 4 — `TimeRoundingTest`: a boundary table

**Why:** R5 (round **up** to 1, 6 or 15 minutes) is the classic place for off-by-one bugs. A data provider
turns the boundary table from the README directly into code.

```php
<?php

// tests/Unit/Support/TimeRoundingTest.php

namespace Tests\Unit\Support;

use App\Support\TimeRounding;
use InvalidArgumentException;
use PHPUnit\Framework\Attributes\DataProvider;
use PHPUnit\Framework\TestCase;

class TimeRoundingTest extends TestCase
{
    #[DataProvider('roundingCases')]
    public function test_rounds_minutes_up_to_the_increment(int $minutes, int $increment, int $expected): void
    {
        $this->assertSame($expected, TimeRounding::roundUp($minutes, $increment));
    }

    public static function roundingCases(): array
    {
        return [
            // 15-minute blocks: the boundary at 15
            'zero stays zero' => [0, 15, 0],
            'one minute becomes a block' => [1, 15, 15],
            'just below a block' => [14, 15, 15],
            'exactly one block stays' => [15, 15, 15],
            'just above a block' => [16, 15, 30],
            'twelve hours is already whole' => [720, 15, 720],
            // 6-minute blocks: the boundary at 6
            'just below six' => [5, 6, 6],
            'exactly six stays' => [6, 6, 6],
            'just above six' => [7, 6, 12],
            // 1-minute blocks: nothing changes
            'one-minute rounding keeps the value' => [7, 1, 7],
            'one-minute rounding keeps zero' => [0, 1, 0],
        ];
    }

    #[DataProvider('invalidInput')]
    public function test_rejects_invalid_input(int $minutes, int $increment): void
    {
        $this->expectException(InvalidArgumentException::class);

        TimeRounding::roundUp($minutes, $increment);
    }

    public static function invalidInput(): array
    {
        return [
            'negative minutes' => [-1, 15],
            'increment zero' => [10, 0],
            'increment not allowed by the rules' => [10, 7],
        ];
    }
}
```

**What happens:**

1. The table has three groups, one per allowed increment. For each increment we test **below, on and
   above** the first boundary. We do not test 29, 30, 31 as well: if the code handles the first boundary
   correctly, it uses the same arithmetic for the second. That is the idea of an equivalence class.
2. `'zero stays zero'` is the row that finds the most common bug. A formula like
   `(intdiv($minutes, $increment) + 1) * $increment` passes 1, 14 and 16 — but turns 0 into 15 and 15
   into 30.
3. `'twelve hours is already whole'` is the upper end of R4 (maximum entry length). Not a boundary of
   the rounding itself, but a realistic large value.
4. The invalid-input table is an equivalence class too: "anything the rules do not allow". Increment 0
   would cause a division by zero; increment 7 is not in the rules (1, 6, 15).

**Check it works:**

```bash
php artisan test --filter=TimeRoundingTest
```

You see 14 passing cases, each with its row name.

Lesson 10's `roundUp()` already has both guard clauses, so these rows pass at once. Prove that they
really test something: comment out the `in_array(...)` check in `app/Support/TimeRounding.php` and run
again. The row `'increment not allowed by the rules'` goes red (and `'increment zero'` fails with a
`DivisionByZeroError` instead of the expected exception). Undo the change.

**Commit:** `test(time-rounding): boundary table for 1, 6 and 15 minute rounding`

---

## Step 5 — The billing strategies, with `Project` objects in memory

**Why:** the strategies take a `Project` model. A model sounds like "database", but a strategy only
*reads* a few attributes (`hourly_rate_cents`, `budget_cap_cents`, `fixed_price_cents`). It never runs a
query. So we can create a `Project` object in memory, without saving it, and the test stays a real unit
test.

### 5a — `HourlyBillingTest`

Create the folder `tests/Unit/Billing`:

```php
<?php

// tests/Unit/Billing/HourlyBillingTest.php

namespace Tests\Unit\Billing;

use App\Billing\HourlyBilling;
use App\Enums\BillingType;
use App\Models\Project;
use App\Support\Money;
use InvalidArgumentException;
use PHPUnit\Framework\TestCase;

class HourlyBillingTest extends TestCase
{
    private HourlyBilling $strategy;

    protected function setUp(): void
    {
        parent::setUp();
        $this->strategy = new HourlyBilling;
    }

    public function test_bills_ninety_minutes_at_sixty_euros_per_hour(): void
    {
        $project = $this->projectWithRate(6000);

        $amount = $this->strategy->calculate($project, 90, Money::fromCents(0));

        $this->assertEquals(Money::fromCents(9000), $amount);
    }

    public function test_bills_nothing_for_zero_minutes(): void
    {
        $project = $this->projectWithRate(6000);

        $amount = $this->strategy->calculate($project, 0, Money::fromCents(0));

        $this->assertEquals(Money::fromCents(0), $amount);
    }

    public function test_rounds_exactly_half_a_cent_up(): void
    {
        // 90.30 € per hour for 1 minute = 150.5 cents
        $project = $this->projectWithRate(9030);

        $amount = $this->strategy->calculate($project, 1, Money::fromCents(0));

        $this->assertEquals(Money::fromCents(151), $amount);
    }

    public function test_rejects_negative_minutes(): void
    {
        $this->expectException(InvalidArgumentException::class);

        $this->strategy->calculate($this->projectWithRate(6000), -1, Money::fromCents(0));
    }

    private function projectWithRate(int $hourlyRateCents): Project
    {
        return (new Project)->forceFill([
            'billing_type' => BillingType::Hourly,
            'hourly_rate_cents' => $hourlyRateCents,
            'rounding_minutes' => 1,
        ]);
    }
}
```

**What happens:**

1. `setUp()` runs **before every test method**. PHPUnit creates a new test object for every test, so each
   test gets a fresh `HourlyBilling`. Nothing leaks from one test to the next.
2. `projectWithRate()` is a small **test data helper**. It hides details that do not matter for the test
   (billing type, rounding) and shows the one that does (the rate). If `Project` gets a new required
   attribute later, you change one helper, not twenty tests.
3. `(new Project)->forceFill([...])` creates a model object and sets its attributes. Nothing is saved;
   no connection is opened. `forceFill()` ignores the `$fillable` list — fine in a test, where we are not
   protecting against malicious form input. (`new Project([...])` also works if all these attributes are
   in `$fillable`.)
4. Setting `'billing_type' => BillingType::Hourly` works because the model's `casts()` maps the column to
   the enum. Casting is pure PHP; it does not need the database.
5. `test_rejects_negative_minutes` covers the guard clause in `HourlyBilling::forMinutes()`. Negative
   minutes can only come from a bug elsewhere; failing loudly is the rule from lesson 10.
6. `test_rounds_exactly_half_a_cent_up` uses a "strange" rate on purpose: 9030 cents per hour ÷ 60 =
   150.5 cents per minute. That is exactly the R3/R6 boundary. The comment explains the number, because
   a reader would otherwise wonder where 9030 comes from.

**Why not `Project::factory()->make()` or `Project::make()`?** Both look harmless, but they are not pure.
`Project::make()` is forwarded to the query builder, which asks for a database connection — in a plain
PHPUnit test there is none, and you get `Call to a member function connection() on null`. Factories need
the Laravel application (and Faker). In a unit test, build the object yourself.

**Watch out for the limits of "no DB".** Reading date attributes (`archived_at`,
`budget_warning_sent_at`) goes through Laravel's `Date` facade, and facades need the application. In a
plain PHPUnit test that fails with *"A facade root has not been set."* The strategies do not read dates,
so we are safe. If a class you want to unit test does, that is a sign it should receive plain values
instead.

**Check it works:** `php artisan test --filter=HourlyBillingTest` → 4 passed.

### 5b — `CappedHourlyBillingTest`

The capped strategy bills the hourly amount but never more than what is left of the cap (R7). Its
behaviour depends on `alreadyBilled`, so the interesting inputs are "how much budget is left".

```php
<?php

// tests/Unit/Billing/CappedHourlyBillingTest.php

namespace Tests\Unit\Billing;

use App\Billing\CappedHourlyBilling;
use App\Billing\HourlyBilling;
use App\Enums\BillingType;
use App\Models\Project;
use App\Support\Money;
use PHPUnit\Framework\TestCase;

class CappedHourlyBillingTest extends TestCase
{
    // 60 €/h, cap 1000 €. 600 minutes = 600 € at the hourly rate.
    private const RATE = 6000;

    private const CAP = 100000;

    public function test_bills_the_hourly_amount_while_far_below_the_cap(): void
    {
        $amount = (new CappedHourlyBilling(new HourlyBilling))->calculate($this->project(), 600, Money::fromCents(0));

        $this->assertEquals(Money::fromCents(60000), $amount);
    }

    public function test_bills_nothing_when_the_cap_is_already_reached(): void
    {
        $amount = (new CappedHourlyBilling(new HourlyBilling))->calculate($this->project(), 600, Money::fromCents(self::CAP));

        $this->assertEquals(Money::fromCents(0), $amount);
    }

    public function test_never_bills_a_negative_amount_when_already_over_the_cap(): void
    {
        $amount = (new CappedHourlyBilling(new HourlyBilling))->calculate($this->project(), 600, Money::fromCents(120000));

        $this->assertEquals(Money::fromCents(0), $amount);
    }

    private function project(): Project
    {
        return (new Project)->forceFill([
            'billing_type' => BillingType::Capped,
            'hourly_rate_cents' => self::RATE,
            'budget_cap_cents' => self::CAP,
            'rounding_minutes' => 1,
        ]);
    }
}
```

**What happens:**

1. The class constants `RATE` and `CAP` plus the comment make the numbers readable: every test uses a
   60 €/h project with a 1000 € cap. `CappedHourlyBilling` needs an `HourlyBilling` in its constructor
   (composition, lesson 08). In a unit test there is no container, so we pass a real `new HourlyBilling`
   ourselves — it is pure, so there is nothing to fake.
2. "Already over the cap" (120000 > 100000) can happen in real life: the owner lowered the cap after
   billing. `max(cap − alreadyBilled, 0)` must stop the result from going negative. Without this test, a
   refactor that drops the `max()` would produce an invoice with a negative line.
3. We have not yet tested the most important boundary: remaining budget **just below, equal to and just
   above** the hourly amount. That is part of your independent work (Task 1) — try it yourself first.

### 5c — `FixedPriceBillingTest`

```php
<?php

// tests/Unit/Billing/FixedPriceBillingTest.php

namespace Tests\Unit\Billing;

use App\Billing\FixedPriceBilling;
use App\Enums\BillingType;
use App\Models\Project;
use App\Support\Money;
use PHPUnit\Framework\TestCase;

class FixedPriceBillingTest extends TestCase
{
    public function test_bills_the_full_price_when_nothing_was_billed_yet(): void
    {
        $amount = (new FixedPriceBilling)->calculate($this->projectWithPrice(250000), 600, Money::fromCents(0));

        $this->assertEquals(Money::fromCents(250000), $amount);
    }

    public function test_bills_only_the_remaining_part_after_a_partial_invoice(): void
    {
        $amount = (new FixedPriceBilling)->calculate($this->projectWithPrice(250000), 600, Money::fromCents(100000));

        $this->assertEquals(Money::fromCents(150000), $amount);
    }

    private function projectWithPrice(int $fixedPriceCents): Project
    {
        return (new Project)->forceFill([
            'billing_type' => BillingType::Fixed,
            'fixed_price_cents' => $fixedPriceCents,
            'rounding_minutes' => 1,
        ]);
    }
}
```

**What happens:** the fixed-price rule (R8) is "the remaining part of the price". The two tests cover the
two normal classes: nothing billed, partly billed. The boundaries (exactly fully billed, over-billed, and
"minutes do not matter") are in Task 1.

**Check it works:**

```bash
php artisan test --filter=Billing
```

`--filter` matches against the full test name including the namespace, so `Billing` runs all three
classes: 9 passed.

**Commit:** `test(billing): unit tests for hourly, capped and fixed-price strategies`

---

## Step 6 — `DueDateCalculatorTest`: dates without the clock

**Why:** R12 is a date rule, and date rules are where tests become non-deterministic (README, section
10). `DueDateCalculator` receives the issue date as an argument, so the test decides "today" — no `now()`
anywhere.

```php
<?php

// tests/Unit/Support/DueDateCalculatorTest.php

namespace Tests\Unit\Support;

use App\Support\DueDateCalculator;
use Carbon\CarbonImmutable;
use PHPUnit\Framework\Attributes\DataProvider;
use PHPUnit\Framework\TestCase;

class DueDateCalculatorTest extends TestCase
{
    #[DataProvider('issueDates')]
    public function test_due_date_is_fourteen_days_later_but_never_on_a_weekend(string $issuedOn, string $expectedDueOn): void
    {
        $dueOn = (new DueDateCalculator)->dueDate(CarbonImmutable::parse($issuedOn));

        $this->assertSame($expectedDueOn, $dueOn->toDateString());
    }

    public static function issueDates(): array
    {
        return [
            'due on a Friday stays on Friday' => ['2026-10-02', '2026-10-16'],
            'due on a Saturday moves to Monday' => ['2026-10-03', '2026-10-19'],
            'due on a Sunday moves to Monday' => ['2026-10-04', '2026-10-19'],
            'due on a Monday stays on Monday' => ['2026-10-05', '2026-10-19'],
            'moving across the new year' => ['2026-12-19', '2027-01-04'],
        ];
    }
}
```

**What happens:**

1. `CarbonImmutable::parse('2026-10-02')` is plain Carbon — a library, not a Laravel facade — so it works
   in a PHPUnit test without the application.
2. We compare `toDateString()` (`'2026-10-16'`) instead of whole Carbon objects. A date string has no time
   and no time zone, so the assertion checks exactly the rule and the failure message is easy to read.
3. Because 14 days are exactly two weeks, the due date falls on the **same weekday** as the issue date.
   That is how the rows were chosen: issue on a Friday → due on a Friday (must not move), Saturday → +2,
   Sunday → +1. A buggy implementation that always adds 2 days on weekends passes the Saturday row and
   fails the Sunday row.
4. The last row crosses a month **and** a year. Date code that does its own day arithmetic often breaks
   there.

**Check it works:** `php artisan test --filter=DueDateCalculatorTest` → 5 passed.

**Commit:** `test(due-date): weekend rule as a data-driven test`

---

## Step 7 — Running tests: the commands you will use

**Why:** in a larger suite you rarely run everything while working on one class.

| Command | What it does |
|---|---|
| `php artisan test` | All suites (Unit + Feature), nice output |
| `php artisan test --testsuite=Unit` | Only `tests/Unit` — the fast ones |
| `php artisan test --filter=MoneyTest` | Only tests whose name matches |
| `php artisan test --filter=test_rejects` | Only methods whose name contains `test_rejects` |
| `php artisan test --stop-on-failure` | Stop at the first red test |
| `./vendor/bin/phpunit` | The same tests with PHPUnit's own, plainer output |

Run the unit suite and look at the duration:

```bash
php artisan test --testsuite=Unit
```

**Check it works:** about 65 test cases, `Duration:` well under one second. That is the speed that lets you
run them after every change. Your editor can do it too: PhpStorm and VS Code (with a PHPUnit extension)
show a green "run" icon next to each test method.

---

## Step 8 — Code coverage

**Why:** coverage shows which lines no test executes (README, section 11). To measure it, PHP needs a
**coverage driver** — an extension that records which lines run. There are two: **Xdebug** (also a
debugger, slower) and **PCOV** (coverage only, fast). You need one of them.

Check what you have:

```bash
php -m | grep -i -E "xdebug|pcov"
```

(On Windows PowerShell: `php -m | Select-String -Pattern "xdebug|pcov"`.)

- If nothing is printed, install one. With Herd, check Herd's documentation on enabling Xdebug. Without
  Herd, install PCOV or Xdebug for your PHP version following their official install pages (see Further
  reading in the README).
- **Xdebug 3** only records coverage in coverage mode. Prefix the command with `XDEBUG_MODE=coverage`.
  PCOV needs no prefix.

Run the unit suite with coverage:

```bash
XDEBUG_MODE=coverage php artisan test --testsuite=Unit --coverage
```

**Check it works:** after the test list you see one line per class in `app/`, for example:

```text
  Billing/CappedHourlyBilling ........................................ 100.0 %
  Billing/FixedPriceBilling .......................................... 100.0 %
  Http/Controllers/ClientController ..................................   0.0 %
  Support/Money ...................................................... 95.2 %
  ...
  Total: 18.4 %
```

The total is low, and that is **correct**: controllers, models and services are not unit tested — they
are covered by feature tests in lessons 12 and 14. Look at the numbers for `Support` and `Billing` only.

For a report where you can click through files and see red and green lines, use PHPUnit directly:

```bash
XDEBUG_MODE=coverage ./vendor/bin/phpunit --testsuite=Unit --coverage-html build/coverage
```

Open `build/coverage/index.html` in the browser. Add `/build` to `.gitignore` — generated files do not
belong in Git.

If you see *"Code coverage driver not available. Did you install Xdebug or PCOV?"*, the extension is not
loaded for the PHP that runs Artisan — see Troubleshooting.

---

## Step 9 — One TDD cycle: `Money::sum()`

**Why:** in lesson 13 an invoice subtotal is the sum of its lines. Today you would write a loop with
`add()` in the invoice code. A `Money::sum()` that takes any number of amounts is clearer and belongs in
`Money`. We add it the TDD way: test first (README, section 12).

**Red.** Decide how the method should look *from the caller's side* and write that down as tests. Add
these to `MoneyTest`, at the end of the class:

```php
// tests/Unit/Support/MoneyTest.php — add inside the class
    public function test_the_sum_of_no_amounts_is_zero(): void
    {
        $this->assertEquals(Money::zero(), Money::sum());
    }

    public function test_the_sum_of_one_amount_is_that_amount(): void
    {
        $this->assertEquals(Money::fromCents(1250), Money::sum(Money::fromCents(1250)));
    }

    public function test_sums_several_amounts(): void
    {
        $total = Money::sum(Money::fromCents(1250), Money::fromCents(399), Money::fromCents(-100));

        $this->assertEquals(Money::fromCents(1549), $total);
    }
```

The three tests follow a classic rule for anything that takes a list: **zero, one, many**. "Zero items"
is the boundary where loops most often go wrong (what is the start value?).

Run them:

```bash
php artisan test --filter=MoneyTest
```

```text
  ⨯ the sum of no amounts is zero
  ⨯ the sum of one amount is that amount
  ⨯ sums several amounts

  Error: Call to undefined method App\Support\Money::sum()
```

Red — for the right reason: the method does not exist yet.

**Green.** Write the simplest code that passes. Add to `app/Support/Money.php`, below `fromEuros()`:

```php
// app/Support/Money.php — add inside the class, below fromEuros()
    public static function sum(Money ...$amounts): self
    {
        $total = self::zero();

        foreach ($amounts as $amount) {
            $total = $total->add($amount);
        }

        return $total;
    }
```

Run `php artisan test --filter=MoneyTest` again → all green.

**What happens:** `Money ...$amounts` is a **variadic** parameter: the method accepts any number of
`Money` arguments, and inside it `$amounts` is an array. With no arguments the array is empty, the loop
does not run and the result is zero. A caller with an array writes `Money::sum(...$lineAmounts)`. Because
each step uses `add()`, the overflow check from lesson 10 still protects the sum.

**Refactor.** With the tests green, look at the code. The loop is fine, but PHP has a function for
"combine a list into one value":

```php
// app/Support/Money.php — sum() after refactoring
    public static function sum(Money ...$amounts): self
    {
        return array_reduce(
            $amounts,
            fn (Money $total, Money $amount): Money => $total->add($amount),
            self::zero(),
        );
    }
```

Run the tests again → still green. That is the point of the refactor step: you changed *how* the method
works, and the tests prove that *what* it does stayed the same. (If you find the loop easier to read,
keep the loop — refactoring means "better", not "shorter".)

**Commit:** `feat(money): add Money::sum()` (the tests and the code in one commit)

---

## Step 10 — A deliberately bad test

**Why:** you learn as much from a bad test as from a good one. This is the bad test from the README,
section 10. **Do not commit it** — type it, run it, and then replace it.

```php
<?php

// tests/Unit/Billing/BadHourlyBillingTest.php — an example of what NOT to do

namespace Tests\Unit\Billing;

use App\Billing\HourlyBilling;
use App\Models\Project;
use App\Support\Money;
use PHPUnit\Framework\TestCase;

class BadHourlyBillingTest extends TestCase
{
    public function test_money(): void
    {
        $minutes = random_int(1, 600);
        $expected = Money::fromCents(intdiv($minutes * 6000 + 30, 60));
        $project = Project::first();

        $result = (new HourlyBilling)->calculate($project, $minutes, Money::fromCents(0));

        $this->assertTrue($result->greaterThanOrEqual(Money::fromCents(0)));
    }
}
```

Run it: `php artisan test --filter=BadHourlyBillingTest`. It **errors** (`Call to a member function
connection() on null`), because `Project::first()` needs a database that a unit test does not have. Even if
it ran in a feature test, it would still be bad:

| Problem | Consequence |
|---|---|
| Name `test_money` | When it fails, nobody knows what broke |
| `random_int()` | A failure cannot be reproduced |
| Expected value uses the production formula | If the formula is wrong, the test is wrong the same way |
| `Project::first()` | Depends on the database and on whatever row is first |
| `$expected` is never used | The real assertion is only "not negative" |
| `greaterThanOrEqual(0)` | A method that always returns zero passes |

The good version of the same idea is `test_bills_ninety_minutes_at_sixty_euros_per_hour` from step 5:
fixed input, a hand-calculated expected value, an in-memory project, an exact assertion.

Delete the file:

```bash
rm tests/Unit/Billing/BadHourlyBillingTest.php
```

---

## Step 11 — Run the tests in CI

**Why:** tests that only run on your laptop are forgotten. CI runs them on a clean machine on every push
(README, section 14).

Open `.github/workflows/ci.yml` from lesson 02. It has one job, `code-style` ("Code style (Pint)"). Add a
second job, `tests`, below it. The complete file then looks like this (the `code-style` job is exactly the
lesson 02 version — keep your own if it differs slightly, for example with the Larastan step):

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  code-style:
    name: Code style (Pint)
    runs-on: ubuntu-latest

    steps:
      - name: Check out code
        uses: actions/checkout@v5

      - name: Set up PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'
          tools: composer:v2
          coverage: none

      - name: Install Composer dependencies
        run: composer install --no-interaction --prefer-dist --no-progress

      - name: Check code style
        run: ./vendor/bin/pint --test

  tests:
    name: Tests
    runs-on: ubuntu-latest

    steps:
      - name: Check out code
        uses: actions/checkout@v5

      - name: Set up PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'
          tools: composer:v2
          coverage: none

      - name: Install Composer dependencies
        run: composer install --no-interaction --prefer-dist --no-progress

      - name: Prepare the environment
        run: |
          cp .env.example .env
          php artisan key:generate

      - name: Run tests
        run: php artisan test
```

**What happens:**

1. The two jobs run **in parallel** on separate fresh Ubuntu machines. If formatting fails, you still see
   whether the tests pass, and the other way round.
2. `shivammathur/setup-php` installs PHP 8.4 with common extensions (including `pdo_sqlite`, which the
   feature tests need). `coverage: none` skips Xdebug/PCOV — faster, and CI does not need coverage.
3. `.env` is not in Git (it contains secrets), so CI creates one from `.env.example` and generates an
   `APP_KEY`. Feature tests need the key for sessions and encryption. The database settings from `.env`
   do not matter: `phpunit.xml` overrides them with SQLite in memory.
4. `php artisan test` exits with a non-zero code if any test fails, which marks the job red.

Before you push, format the new test files — Pint checks `tests/` too:

```bash
./vendor/bin/pint
php artisan test
git add .
git commit -m "ci: run the test suite on every push"
git push
```

**Check it works:** on GitHub, open the **Actions** tab. The latest run shows two green jobs, **Code style
(Pint)** and **Tests**. Open **Tests** → "Run tests" and you see the same list of passing tests as on your machine.

**Commit:** `ci: run the test suite on every push`

---

## Troubleshooting

| Error or symptom | Cause | Fix |
|---|---|---|
| `RuntimeException: A facade root has not been set.` | A unit test (extends `PHPUnit\Framework\TestCase`) reached a facade or helper: `now()`, `config()`, `Log::`, a date cast on a model | Keep the code under test pure (pass values in). If it really needs Laravel, it is a feature test: move it to `tests/Feature` and extend `Tests\TestCase` |
| `Error: Call to a member function connection() on null` | `Project::make()`, `Project::first()`, `Project::query()` or a relation was called in a unit test | Build the model with `(new Project)->forceFill([...])`; never query in a unit test |
| `Illuminate\Database\Eloquent\MassAssignmentException: Add [budget_cap_cents] to fillable property` | `new Project([...])` with an attribute that is not in `$fillable` | Use `forceFill()` in tests |
| `No tests found.` or a new test file is ignored | File name does not end in `Test.php`, or class name differs from file name, or method name does not start with `test` | Rename: `MoneyTest.php` with `class MoneyTest`, methods `test_...` |
| `The data provider specified for ... is invalid` / `Data Provider method ... is not static` | Provider not `public static`, or the name in the attribute is misspelt | `public static function cases(): array` and `#[DataProvider('cases')]` |
| Data provider silently not used; test fails with `ArgumentCountError: Too few arguments` | Doc comment `/** @dataProvider cases */` on PHPUnit 12 | Use the `#[DataProvider('cases')]` attribute and import `PHPUnit\Framework\Attributes\DataProvider` |
| `Metadata found in doc-comment for method ... is deprecated` | Same thing on PHPUnit 11 | Same fix |
| `This test did not perform any assertions` (risky) | A test with no `assert...` and no `expectException` | Add the assertion — a test without one checks nothing |
| `Code coverage driver not available. Did you install Xdebug or PCOV?` | No coverage extension loaded, or Xdebug without coverage mode | Install PCOV or Xdebug; with Xdebug run `XDEBUG_MODE=coverage php artisan test --coverage`. Check with `php -m` that it is loaded for the same PHP binary |
| CI job **Code style (Pint)** red after adding tests | Test files are not formatted | Run `./vendor/bin/pint` locally and commit |
| CI job **Tests** red: `MissingAppKeyException` / `No application encryption key has been specified` | `.env` missing in CI | The "Prepare the environment" step (copy `.env.example`, `key:generate`) |
| A test passes locally but fails in CI (or on another day) | It depends on local data, the current date or the time zone | Remove the dependency: fixed dates, no DB in unit tests |

---

## Recap

- `tests/Unit` tests extend `PHPUnit\Framework\TestCase`; they do not boot Laravel and do not touch the
  database. That is what makes them fast and isolated.
- New files: `tests/Unit/Support/MoneyTest.php`, `TimeRoundingTest.php`, `DueDateCalculatorTest.php`,
  `tests/Unit/Billing/HourlyBillingTest.php`, `CappedHourlyBillingTest.php`, `FixedPriceBillingTest.php`.
- `#[DataProvider('name')]` with a `public static` method turns a boundary table into one test with many
  named rows.
- `expectException()` goes **before** the act; the test passes only if the exception is thrown.
- `Project` objects for strategy tests are built with `(new Project)->forceFill([...])` — fine because no
  query runs. `Project::make()` and factories are not pure.
- Coverage needs Xdebug or PCOV; `php artisan test --coverage` shows the per-class numbers, PHPUnit's
  `--coverage-html` a clickable report.
- `Money::sum()` was added test-first (red → green → refactor), with zero, one and many amounts.
- `.github/workflows/ci.yml` has a `tests` job ("Tests") next to `code-style` from lesson 02.

---

## Independent work — solutions

<details>
<summary><strong>Task 1 (Basic)</strong> — 15+ meaningful unit tests, every billing type at its boundaries</summary>

After the walkthrough you already have more than 15 test cases. What Task 1 adds is the **boundaries of
the strategies** that class did not cover.

**Capped:** the central boundary is "remaining budget compared with the hourly amount". With a 1000 € cap
and 600 minutes at 60 €/h (= 600 €), the remaining budget is exactly 600 € when 400 € is already billed.
Add this data-driven test to `CappedHourlyBillingTest`:

```php
// tests/Unit/Billing/CappedHourlyBillingTest.php — add inside the class,
// and add "use PHPUnit\Framework\Attributes\DataProvider;" to the imports
    #[DataProvider('remainingBudget')]
    public function test_bills_the_smaller_of_hourly_amount_and_remaining_budget(int $alreadyBilledCents, int $expectedCents): void
    {
        $amount = (new CappedHourlyBilling(new HourlyBilling))->calculate($this->project(), 600, Money::fromCents($alreadyBilledCents));

        $this->assertEquals(Money::fromCents($expectedCents), $amount);
    }

    public static function remainingBudget(): array
    {
        // Hourly amount for 600 minutes is always 600.00 €.
        return [
            'remaining 600.01 € is more than hourly' => [39999, 60000],
            'remaining exactly 600.00 € equals hourly' => [40000, 60000],
            'remaining 599.99 € is less than hourly' => [40001, 59999],
            'one cent left' => [99999, 1],
        ];
    }
```

**Fixed price:**

```php
// tests/Unit/Billing/FixedPriceBillingTest.php — add inside the class,
// and add "use PHPUnit\Framework\Attributes\DataProvider;" to the imports
    #[DataProvider('alreadyBilled')]
    public function test_bills_what_is_left_of_the_price_but_never_less_than_zero(int $alreadyBilledCents, int $expectedCents): void
    {
        $amount = (new FixedPriceBilling)->calculate($this->projectWithPrice(250000), 600, Money::fromCents($alreadyBilledCents));

        $this->assertEquals(Money::fromCents($expectedCents), $amount);
    }

    public static function alreadyBilled(): array
    {
        return [
            'one cent left' => [249999, 1],
            'exactly fully billed' => [250000, 0],
            'over-billed' => [300000, 0],
        ];
    }

    public function test_the_number_of_minutes_does_not_change_a_fixed_price(): void
    {
        $strategy = new FixedPriceBilling;
        $project = $this->projectWithPrice(250000);

        $this->assertEquals(
            $strategy->calculate($project, 0, Money::fromCents(0)),
            $strategy->calculate($project, 6000, Money::fromCents(0)),
        );
    }
```

**Hourly:**

```php
// tests/Unit/Billing/HourlyBillingTest.php — add inside the class
    public function test_rounds_just_below_half_a_cent_down(): void
    {
        // 90.29 € per hour for 1 minute = 150.483 cents
        $project = $this->projectWithRate(9029);

        $amount = $this->strategy->calculate($project, 1, Money::fromCents(0));

        $this->assertEquals(Money::fromCents(150), $amount);
    }

    public function test_ignores_what_was_already_billed(): void
    {
        $project = $this->projectWithRate(6000);

        $amount = $this->strategy->calculate($project, 60, Money::fromCents(100000));

        $this->assertEquals(Money::fromCents(6000), $amount);
    }
```

The last fixed-price test compares two results of the code with each other. That is an exception to
"write expected values by hand", and it is fine here: the behaviour *is* "the result does not depend on
minutes". An extra `assertEquals(Money::fromCents(250000), ...)` would make it even clearer.

**Counting.** A good count lists behaviours, not rows:

| Class | Behaviours tested |
|---|---|
| `Money` | add, subtract, multiply, immutability, parse valid, parse invalid, half-up percentage (incl. negative), isZero, isNegative, gte equal, gte smaller, format, sum ×3 |
| `TimeRounding` | round up at 1/6/15, invalid input |
| `HourlyBilling` | normal, zero, negative minutes rejected, half cent up, just below half, ignores already billed |
| `CappedHourlyBilling` | far below cap, remaining vs hourly (table), cap reached, over cap |
| `FixedPriceBilling` | full, partial, one cent left / exactly billed / over-billed, minutes irrelevant |
| `DueDateCalculator` | weekend rule (table) |

That is well above 15 meaningful tests.

**Commit:** `test(billing): boundary tables for capped and fixed-price strategies`

</details>

<details>
<summary><strong>Task 2 (Basic)</strong> — extract <code>BudgetMonitor::reachedThreshold()</code> and test it with a boundary table</summary>

**1. The pure method.** Lesson 10 already compares exactly, but the comparison is buried in `check()`
next to the database and the notifier. Add a static method to `app/Services/BudgetMonitor.php`:

```php
// app/Services/BudgetMonitor.php — add inside the class, below check()
    /**
     * True when $value is at least 80 % of $cap (R9). Exact integer comparison, no rounding.
     */
    public static function reachedThreshold(Money $value, Money $cap): bool
    {
        return $value->multiply(100)->greaterThanOrEqual($cap->multiply(self::WARNING_THRESHOLD_PERCENT));
    }
```

Then, in `check()`, replace the lesson 10 comparison

```php
        if (! $value->multiply(100)->greaterThanOrEqual($cap->multiply(self::WARNING_THRESHOLD_PERCENT))) {
```

with a call to the new method:

```php
// app/Services/BudgetMonitor.php — inside check()
        if (! self::reachedThreshold($value, $cap)) {
            return;
        }
```

**Why exact comparison and not `percentageBp(8000)`?** (This is the trap the task hint points at.)
`$cap->percentageBp(8000)` rounds. For a cap of 9,99 € it gives 799.2 → **7,99 €**. A value of 7,99 € is
only 79.98 % of the cap, but `7,99 € >= 7,99 €` would warn. Comparing `value × 100` with `cap × 80` never
divides, so nothing is rounded: `799 × 100 = 79 900` is less than `999 × 80 = 79 920` → no warning.
Correct. The test below has a row for exactly this case, so nobody can "simplify" the method later
without a red test.

**Why static?** The method uses nothing from the object — no notifier, no database. `static` says that
openly, and the test can call it without building a `BudgetMonitor`.

**2. The test.** It is a unit test (no Laravel, no database), even though `BudgetMonitor` itself is not
pure:

```php
<?php

// tests/Unit/Services/BudgetMonitorThresholdTest.php

namespace Tests\Unit\Services;

use App\Services\BudgetMonitor;
use App\Support\Money;
use PHPUnit\Framework\Attributes\DataProvider;
use PHPUnit\Framework\TestCase;

class BudgetMonitorThresholdTest extends TestCase
{
    #[DataProvider('thresholdCases')]
    public function test_decides_whether_80_percent_of_the_cap_is_reached(int $valueCents, int $capCents, bool $expected): void
    {
        $reached = BudgetMonitor::reachedThreshold(Money::fromCents($valueCents), Money::fromCents($capCents));

        $this->assertSame($expected, $reached);
    }

    public static function thresholdCases(): array
    {
        return [
            'nothing logged (0 %)' => [0, 10000, false],
            'one cent below (79.99 %)' => [7999, 10000, false],
            'exactly 80 %' => [8000, 10000, true],
            'one cent above (80.01 %)' => [8001, 10000, true],
            'the whole cap (100 %)' => [10000, 10000, true],
            'over the cap (120 %)' => [12000, 10000, true],
            'odd cap 9.99 €: 7.99 € is only 79.98 %' => [799, 999, false],
            'odd cap 9.99 €: 8.00 € is 80.08 %' => [800, 999, true],
        ];
    }
}
```

**3. Check.** `php artisan test --filter=BudgetMonitorThresholdTest` → 8 passed. Then log time on a
capped project in the browser until it crosses 80 % and check Mailpit (`http://localhost:8025`): the
warning still arrives, once.

**Commit:** `refactor(budget): extract reachedThreshold() and cover it with a boundary table`

</details>

<details>
<summary><strong>Task 3 (Intermediate)</strong> — coverage report and <code>docs/testing.md</code></summary>

Generate the report:

```bash
XDEBUG_MODE=coverage ./vendor/bin/phpunit --testsuite=Unit --coverage-html build/coverage
```

Make sure `.gitignore` contains:

```text
# .gitignore — add
/build
```

Example `docs/testing.md` (use your own numbers and your own reasons):

```markdown
<!-- docs/testing.md -->
# Testing

## Running the tests

- All tests: `php artisan test`
- Only unit tests (fast, no database): `php artisan test --testsuite=Unit`
- Coverage report (needs PCOV or Xdebug):
  `XDEBUG_MODE=coverage ./vendor/bin/phpunit --testsuite=Unit --coverage-html build/coverage`,
  then open `build/coverage/index.html`.

## Coverage of the domain code (unit tests only)

| Area | Lines |
|---|---|
| `app/Support` | 97 % |
| `app/Billing` | 100 % |

## What is not unit tested, and why

- **`BillingStrategyResolver`.** It is a `match` on the billing type that returns one of three
  objects. A unit test would repeat the `match` line by line. It is covered by the feature test of
  the project page, which shows "billable so far" for each billing type.
- **`BudgetMonitor::check()`.** It reads time entries from the database, sends a notification and
  writes `budget_warning_sent_at`. Its decision (80 %) is extracted into `reachedThreshold()` and unit
  tested; the rest is tested with a mocked notifier in a feature test (lesson 12).
- **Controllers and Form Requests.** They contain no calculations; their job is wiring and
  validation rules. Feature tests (lessons 12 and 14) post real forms against them.
- **`Money::formatLocale()`.** `format()` itself is tested (including the thousands separator); the
  optional `formatLocale()` from the lesson 10 advanced task depends on the ICU data of the `intl`
  extension and is not tested yet.
```

What makes this a good judgement: every "not tested" item has a **reason** and says **where else** it is
covered (or admits that it is not). "No time" is not a reason.

**Commit:** `docs(testing): coverage report and what is left untested`

</details>

<details>
<summary><strong>Task 4 (Advanced)</strong> — mutation testing with Infection</summary>

Infection changes your code in small ways (**mutants**) and runs the tests against each change. A mutant
that makes a test fail is **killed** (good). A mutant that all tests survive shows a line whose behaviour
no test really checks.

Install it and make sure a coverage driver is available (Infection needs coverage to know which tests to
run for which line):

```bash
composer require --dev infection/infection
```

Create the configuration in the project root, limited to the pure code and the unit suite:

```json5
// infection.json5
{
    "$schema": "vendor/infection/infection/resources/schema.json",
    "source": {
        "directories": ["app/Support", "app/Billing"]
    },
    "logs": {
        "text": "build/infection.log",
        "html": "build/infection.html"
    },
    "testFrameworkOptions": "--testsuite=Unit"
}
```

(If you run Infection without this file, it asks the same questions interactively and writes the file for
you.) Composer may ask to allow the `infection/extension-installer` plugin — answer yes.

Run it:

```bash
XDEBUG_MODE=coverage ./vendor/bin/infection --threads=4
```

**Check it works:** at the end Infection prints counts of killed and escaped mutants and a **Mutation
Score Indicator (MSI)**. Open `build/infection.html` to see every escaped mutant as a diff.

**An example you are likely to see.** Infection changes number literals by one. In
`Money::divideHalfUp()` it turns `intdiv($divisor, 2)` into `intdiv($divisor, 3)`, so ÷ 60 adds 20
instead of 30. Suppose your only rounding tests were 90 minutes and 0 minutes at 60 €/h: both still give
the right result — the mutant **escapes**. The half-cent rows kill it: 1 minute at 9030 (150.5 → must be
151; adding 20 gives 150), and the "exactly half" rows of the percentage table in `MoneyTest`. Because
VAT and the hourly amount share one rounding function, the tests of each one protect the other.

**An equivalent mutant** changes the code without changing behaviour — for example `$minutes === 0 ?
0 : ...` changed to `$minutes <= 0 ? 0 : ...` when negative minutes are already rejected earlier. No test
can kill it; write down why it is harmless instead of adding a meaningless test.

Add a short section to `docs/testing.md`: the MSI, one escaped mutant, and what you did about it.

**Commit:** `test: add Infection mutation testing for Support and Billing`

</details>
