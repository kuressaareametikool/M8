# 14 — Integration tests and data integrity · Laravel

[← Concepts](README.md) · Starting point: end of lesson 13 · Estimated time in class: 2 h

## What we build today

- `tests/Feature/ClientCrudTest.php` — client create / update / delete over HTTP
- `tests/Feature/InvoiceFlowTest.php` — the whole invoice flow over HTTP: numbers `2026-0001` and
  `2026-0002`, totals, linked entries, overlap message, draft deletion, "sent cannot be deleted"
- `tests/Feature/TimeSheetQueryCountTest.php` — the time sheet runs a fixed number of queries
- An optional PostgreSQL test database `billable_test` and `phpunit.pgsql.xml`
- An Artisan command `invoices:race-demo` that **shows** the duplicate-number race with two processes
- The `invoice_sequences` table and a new `InvoiceNumberGenerator` that locks the year's row
- Row locks on the entries being invoiced, so the same hours cannot be billed twice
- A feature test for the new numbering, including the first invoice of a new year

**Final result:** `php artisan test` runs unit **and** feature tests; running the race demo twice in
parallel gives `2026-000N` and `2026-000N+1` instead of a crash.

---

## Step 1 — Feature tests for clients

**Why:** a feature test sends a real HTTP request through routing, middleware, the form request, the
controller, the service and the database, then checks the response **and** what was stored. It is the
closest thing to a user clicking through the app.

```php
<?php

// tests/Feature/ClientCrudTest.php

namespace Tests\Feature;

use App\Models\Client;
use App\Models\Project;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class ClientCrudTest extends TestCase
{
    use RefreshDatabase;

    public function test_a_client_can_be_created(): void
    {
        $response = $this->post(route('clients.store'), [
            'name' => 'Saare Web OÜ',
            'email' => 'arved@saareweb.ee',
            'vat_number' => 'EE101234567',
        ]);

        $client = Client::query()->where('email', 'arved@saareweb.ee')->firstOrFail();
        $response->assertRedirect(route('clients.show', $client));
        $response->assertSessionHasNoErrors();
        $this->assertDatabaseHas('clients', ['name' => 'Saare Web OÜ', 'vat_number' => 'EE101234567']);
    }

    public function test_invalid_input_is_rejected_and_nothing_is_saved(): void
    {
        $response = $this->from(route('clients.create'))->post(route('clients.store'), [
            'name' => '',
            'email' => 'not-an-email',
        ]);

        $response->assertRedirect(route('clients.create'));
        $response->assertSessionHasErrors(['name', 'email']);
        $this->assertDatabaseCount('clients', 0);
    }

    public function test_the_email_must_be_unique(): void
    {
        Client::factory()->create(['email' => 'arved@saareweb.ee']);

        $this->post(route('clients.store'), ['name' => 'Copy', 'email' => 'arved@saareweb.ee'])
            ->assertSessionHasErrors('email');

        $this->assertDatabaseCount('clients', 1);
    }

    public function test_a_client_can_be_updated(): void
    {
        $client = Client::factory()->create();

        $this->put(route('clients.update', $client), [
            'name' => 'New name',
            'email' => $client->email,
        ])->assertRedirect(route('clients.show', $client));

        $this->assertSame('New name', $client->fresh()->name);
    }

    public function test_a_client_without_projects_can_be_deleted(): void
    {
        $client = Client::factory()->create();

        $this->delete(route('clients.destroy', $client))->assertRedirect(route('clients.index'));

        $this->assertModelMissing($client);
    }

    public function test_a_client_with_projects_is_not_deleted(): void
    {
        $client = Client::factory()->create();
        Project::factory()->for($client)->create();

        $this->delete(route('clients.destroy', $client))->assertSessionHas('error');

        $this->assertModelExists($client);
    }
}
```

**What happens:**

1. `$this->post(route('clients.store'), [...])` sends a `POST /clients` inside the test process — no
   web server, but the full Laravel request lifecycle (lesson 01). CSRF checks are switched off
   automatically while tests run.
2. `assertRedirect(...)` checks the status is `302` and the `Location` header. `assertSessionHasErrors`
   checks the validation errors that the form request put in the session.
3. `from(route('clients.create'))` sets the "previous URL", so `back()` after a failed validation
   redirects there — exactly what happens in a browser.
4. `assertDatabaseHas` / `assertDatabaseCount` / `assertModelMissing` look into the database. A test
   that only checks the redirect would pass even if nothing was saved.
5. The last test checks the **rule** "a client with projects cannot be deleted" from lesson 05/07, through
   HTTP, including the flashed `error`.

If your lesson 05 messages or redirects differ (for example `clients.index` after update), change the
assertion — the test documents **your** behaviour.

**Check it works:** `php artisan test --filter=ClientCrudTest` → `Tests: 6 passed`.

**Commit:** `test(clients): feature tests for client CRUD`

---

## Step 2 — The invoice flow over HTTP

**Why:** lesson 13 tested `InvoiceGenerator` by calling it. Now we test what the **user** does: open
the form, post it, land on the invoice, send it, try to delete it.

```php
<?php

// tests/Feature/InvoiceFlowTest.php

namespace Tests\Feature;

use App\Enums\BillingType;
use App\Enums\InvoiceStatus;
use App\Models\Client;
use App\Models\Invoice;
use App\Models\Project;
use App\Models\TimeEntry;
use Carbon\CarbonImmutable;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Testing\TestResponse;
use Tests\TestCase;

final class InvoiceFlowTest extends TestCase
{
    use RefreshDatabase;

    private Client $client;

    private Project $project;

    protected function setUp(): void
    {
        parent::setUp();

        $this->travelTo(CarbonImmutable::parse('2026-09-30 12:00'));

        $this->client = Client::factory()->create(['name' => 'Saare Web OÜ']);
        $this->project = Project::factory()->for($this->client)->create([
            'name' => 'Website redesign',
            'billing_type' => BillingType::Hourly,
            'hourly_rate_cents' => 6000,
            'fixed_price_cents' => null,
            'budget_cap_cents' => null,
            'rounding_minutes' => 15,
        ]);
    }

    public function test_generating_an_invoice_over_http(): void
    {
        $first = $this->entry('2026-09-28 09:00', '2026-09-28 10:10');
        $second = $this->entry('2026-09-29 13:00', '2026-09-29 14:00');

        $response = $this->generate();

        $invoice = Invoice::query()->sole();
        $response->assertRedirect(route('invoices.show', $invoice));
        $this->assertSame('2026-0001', $invoice->number);
        $this->assertSame(InvoiceStatus::Draft, $invoice->status);
        $this->assertSame([13500, 3240, 16740], [$invoice->subtotal_cents, $invoice->vat_cents, $invoice->total_cents]);
        $this->assertDatabaseHas('invoice_lines', [
            'invoice_id' => $invoice->id,
            'description' => 'Website redesign — 2 h 15 min',
            'minutes' => 135,
            'amount_cents' => 13500,
        ]);
        $this->assertSame($invoice->id, $first->fresh()->invoice_id);
        $this->assertSame($invoice->id, $second->fresh()->invoice_id);

        $this->get(route('invoices.show', $invoice))
            ->assertOk()
            ->assertSee('2026-0001')
            ->assertSee('Website redesign — 2 h 15 min');
    }

    public function test_the_second_invoice_gets_the_next_number(): void
    {
        $this->entry('2026-09-28 09:00', '2026-09-28 10:00');
        $this->generate();

        $this->entry('2026-09-29 09:00', '2026-09-29 10:00');
        $this->generate();

        $this->assertSame(['2026-0001', '2026-0002'], Invoice::query()->orderBy('id')->pluck('number')->all());
    }

    public function test_overlapping_entries_are_rejected_with_both_entries_named(): void
    {
        $this->entry('2026-09-29 13:00', '2026-09-29 17:00', 'API integration');
        $this->entry('2026-09-29 13:30', '2026-09-29 14:00', 'Client call');

        $response = $this->from(route('invoices.create'))->generate();

        $response->assertRedirect(route('invoices.create'));
        $response->assertSessionHasErrors('invoice');
        $message = session('errors')->first('invoice');
        $this->assertStringContainsString('API integration', $message);
        $this->assertStringContainsString('Client call', $message);
        $this->assertDatabaseCount('invoices', 0);
    }

    public function test_deleting_a_draft_releases_its_entries(): void
    {
        $entry = $this->entry('2026-09-28 09:00', '2026-09-28 10:00');
        $this->generate();
        $invoice = Invoice::query()->sole();

        $this->delete(route('invoices.destroy', $invoice))->assertRedirect(route('invoices.index'));

        $this->assertModelMissing($invoice);
        $this->assertDatabaseCount('invoice_lines', 0);
        $this->assertNull($entry->fresh()->invoice_id);
    }

    public function test_a_sent_invoice_cannot_be_deleted(): void
    {
        $entry = $this->entry('2026-09-28 09:00', '2026-09-28 10:00');
        $this->generate();
        $invoice = Invoice::query()->sole();
        $this->post(route('invoices.send', $invoice))->assertRedirect(route('invoices.show', $invoice));

        $this->from(route('invoices.show', $invoice))
            ->delete(route('invoices.destroy', $invoice))
            ->assertRedirect(route('invoices.show', $invoice))
            ->assertSessionHas('error');

        $this->assertModelExists($invoice);
        $this->assertSame(InvoiceStatus::Sent, $invoice->fresh()->status);
        $this->assertSame($invoice->id, $entry->fresh()->invoice_id);
    }

    private function generate(): TestResponse
    {
        return $this->post(route('invoices.store'), [
            'client_id' => $this->client->id,
            'up_to' => '2026-09-30',
        ]);
    }

    private function entry(string $start, string $end, string $description = 'Work'): TimeEntry
    {
        return TimeEntry::factory()
            ->for($this->project)
            ->between($start, $end)
            ->create(['description' => $description]);
    }
}
```

**What happens:**

1. `travelTo()` fixes "today" at 30 September 2026, so the number is `2026-…` and the date validation
   `before_or_equal:today` accepts `2026-09-30`.
2. `generate()` posts the same two fields the create form sends. The helper keeps every test short and
   makes the "user action" obvious. `entry()` uses the `TimeEntryFactory` from lesson 13 with fixed
   times, so the totals are predictable.
3. `Invoice::query()->sole()` returns the only invoice and **fails** if there are zero or several — a
   built-in assertion.
4. The first test follows the redirect by hand with `get(...)` and checks the page shows the number and
   the line: the view works, not just the database.
5. The overlap test checks the redirect back to the form, the error key `invoice` and the message
   content, and that nothing was saved.
6. The last two tests check R13 from both sides: a draft can be deleted and its entries are released; a
   sent invoice is refused, and nothing changes — the invoice, its status and the link to its entries.
7. Every test starts with an empty database (`RefreshDatabase`), so the first invoice is always
   `2026-0001`, whatever order the tests run in.

**Check it works:** `php artisan test --filter=InvoiceFlowTest` → `Tests: 5 passed`. Then break
something on purpose — for example, in `InvoiceGenerator::buildLines()` use `$entry->durationMinutes()`
directly instead of rounding it with `TimeRounding::roundUp()` — and watch
`test_generating_an_invoice_over_http` fail (130 minutes and 13,000 cents instead of 135 and 13,500).
Put it back.

**Commit:** `test(invoices): end-to-end invoice flow over HTTP`

---

## Step 3 — A query-count test for the time sheet

**Why:** in lesson 06 you fixed an N+1 on the time sheet. Nothing stops a future change from bringing
it back — unless a test counts the queries.

If you did the lesson 06 advanced task, you already have a file with this name and class. Replace it with
this version (it uses the route name and allows for the time sheet as it is after lesson 07).

```php
<?php

// tests/Feature/TimeSheetQueryCountTest.php

namespace Tests\Feature;

use App\Models\TimeEntry;
use Illuminate\Database\Eloquent\Factories\Sequence;
use Illuminate\Foundation\Testing\RefreshDatabase;
use PHPUnit\Framework\Attributes\TestWith;
use Tests\TestCase;

final class TimeSheetQueryCountTest extends TestCase
{
    use RefreshDatabase;

    /** How many queries the time sheet needs — the same number for any number of entries. */
    private const QUERIES = 6;

    #[TestWith([5])]
    #[TestWith([30])]
    public function test_the_time_sheet_query_count_does_not_grow_with_the_number_of_entries(int $entries): void
    {
        TimeEntry::factory()
            ->count($entries)
            ->sequence(fn (Sequence $sequence) => [
                'started_at' => now()->subDays($sequence->index + 1)->setTime(9, 0),
                'ended_at' => now()->subDays($sequence->index + 1)->setTime(10, 0),
            ])
            ->create();

        $this->expectsDatabaseQueryCount(self::QUERIES);

        $this->get(route('time-entries.index'))->assertOk();
    }
}
```

**What happens:**

1. `TimeEntryFactory` (lesson 13) gives every entry its **own** project, and through `ProjectFactory`
   its own client. If the page lazy-loaded `$entry->project` or `$entry->project->client`, each extra
   entry would add queries. `sequence()` puts the entries on different days, so they never overlap.
2. `#[TestWith([5])]` and `#[TestWith([30])]` run the same test twice, with 5 and with 30 entries.
   Both runs expect **the same** number of queries — that is the "does not grow" rule.
3. `expectsDatabaseQueryCount()` is built into Laravel's `TestCase`. It counts every query from the
   moment you call it until the test ends, and then fails with
   `Expected 6 database queries on the [sqlite] connection. 7 occurred.` That is why we call it
   **after** creating the data: the inserts must not be counted.
4. `QUERIES = 6` fits the time sheet as built in lessons 06–07 (entries, projects, clients, tags, plus
   the projects and clients for the "log time" form). Your page may need a different number: run the
   test once, read the "N occurred" in the message, check that N makes sense for what the page shows,
   and set the constant. If the 5-entry run and the 30-entry run report different numbers, you have N+1.
5. If your time sheet shows only the current week, make sure the created entries fall inside what the
   page shows (here: the last 30 days). If it paginates, keep both sizes on one page.
6. `Model::shouldBeStrict()` from lesson 06 (its `preventLazyLoading` guard) already throws in tests
   when lazy loading happens. This test also catches **other** N+1 shapes, such as a query inside a
   loop in a helper or a view.

> Lesson 06 counted queries by hand with `DB::listen()`. That still works and shows what happens
> underneath; `expectsDatabaseQueryCount()` is the same listener, already written and tested for you.

**Check it works:** run it, then remove `'project.client'` (or whatever you eager-load) from the time
sheet query. The test fails with `LazyLoadingViolationException` or with the query-count message.

**Commit:** `test(time-entries): guard the time sheet against N+1`

---

## Step 4 — (Optional) Run tests on PostgreSQL

**Why:** `phpunit.xml` runs feature tests on SQLite in memory (lessons 06 and 11). That stays the **default**:
it is fast, needs no database service in CI, and — as lesson 06 warned — it means `RefreshDatabase`
can never wipe your development PostgreSQL by accident. But SQLite is not the database you use in
production, and everything in Steps 5–8 is about PostgreSQL **row locks**, which SQLite ignores
(`lockForUpdate()` compiles to nothing). The race demo below runs against your development database,
so it works without this step. To run the **test suite** on PostgreSQL as well — required for any test
that is about locking or concurrency — add a separate test database.

Create it once in the running container:

```bash
docker compose exec postgres createdb -U billable billable_test
```

To have it created automatically on a **fresh** volume, add an init script and mount it:

```sql
-- docker/postgres/init/01-create-test-database.sql
CREATE DATABASE billable_test OWNER billable;
```

```yaml
# compose.yaml (add one line under services → postgres → volumes)
      - ./docker/postgres/init:/docker-entrypoint-initdb.d:ro
```

The official image runs files in `/docker-entrypoint-initdb.d` **only when the data folder is empty**.
Your existing volume already has data, so the script will not run until `docker compose down -v` (which
deletes all your development data). That is why we used `createdb` above.

Now copy `phpunit.xml` to `phpunit.pgsql.xml` and change only the database lines inside `<php>`:

```xml
<!-- phpunit.pgsql.xml (inside <php>; replace the two sqlite lines) -->
<env name="DB_CONNECTION" value="pgsql"/>
<env name="DB_HOST" value="127.0.0.1"/>
<env name="DB_PORT" value="5432"/>
<env name="DB_DATABASE" value="billable_test"/>
<env name="DB_USERNAME" value="billable"/>
<env name="DB_PASSWORD" value="secret"/>
```

```bash
php artisan test --configuration=phpunit.pgsql.xml
```

**What happens:**

1. `--configuration` is passed on to PHPUnit, which reads the other file. `phpunit.xml` stays the
   default, so CI keeps working without a database service.
2. `RefreshDatabase` runs `migrate:fresh` on `billable_test` once per run, then wraps each test in a
   transaction that is rolled back. The first run is slower; later tests are fast.
3. Never point tests at `billable` itself: `RefreshDatabase` would drop all your development data.

**Check it works:** all tests pass with both configurations. If one passes on SQLite and fails on
PostgreSQL (or the other way round), you have found a dialect difference — write it down in your README.

**Commit:** `test: optional PostgreSQL test configuration`

---

## Step 5 — See the race with your own eyes

**Why:** you only believe a race condition after you have seen one. Two requests in the same
millisecond are hard to produce by hand, so we make the dangerous window **longer**: take a number,
sleep, then save.

```bash
php artisan make:command InvoiceRaceDemo
```

```php
<?php

// app/Console/Commands/InvoiceRaceDemo.php

namespace App\Console\Commands;

use App\Enums\InvoiceStatus;
use App\Invoicing\InvoiceNumberGenerator;
use App\Models\Client;
use App\Models\Invoice;
use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;

/**
 * Development tool: takes an invoice number, waits, then saves an empty invoice with it.
 * Run two copies at the same time to see whether numbering is safe under concurrency.
 */
class InvoiceRaceDemo extends Command
{
    protected $signature = 'invoices:race-demo {--sleep=3 : Seconds to wait between taking the number and saving}';

    protected $description = 'Demonstrate (or disprove) the duplicate invoice number race';

    public function handle(InvoiceNumberGenerator $numbers): int
    {
        $client = Client::query()->firstOrFail();
        $pid = getmypid();

        $invoice = DB::transaction(function () use ($numbers, $client, $pid): Invoice {
            $this->line("[{$pid}] asking for a number…");
            $number = $numbers->next((int) today()->year);
            $this->line("[{$pid}] got {$number}, sleeping {$this->option('sleep')} s");

            sleep((int) $this->option('sleep'));

            return $client->invoices()->create([
                'number' => $number,
                'status' => InvoiceStatus::Draft,
                'issued_on' => today(),
                'due_on' => today()->addDays(14),
                'vat_rate_bp' => 2400,
                'subtotal_cents' => 0,
                'vat_cents' => 0,
                'total_cents' => 0,
            ]);
        });

        $this->info("[{$pid}] saved {$invoice->number}");

        return self::SUCCESS;
    }
}
```

Run two copies **in parallel** against your development database (macOS, Linux, Git Bash):

```bash
php artisan invoices:race-demo & php artisan invoices:race-demo & wait
```

On Windows PowerShell, open two terminals and start the command in both within three seconds.

**Expected output with lesson 13's "max + 1":**

```
[4121] asking for a number…
[4122] asking for a number…
[4121] got 2026-0003, sleeping 3 s
[4122] got 2026-0003, sleeping 3 s
[4121] saved 2026-0003

   Illuminate\Database\UniqueConstraintViolationException

  SQLSTATE[23505]: Unique violation: 7 ERROR:  duplicate key value violates unique constraint "invoices_number_unique"
DETAIL:  Key (number)=(2026-0003) already exists.
```

**What happens:**

1. Each process opens its own connection and its own transaction.
2. Both read `MAX(number)` while neither has committed. In READ COMMITTED they see the same value and
   compute the same next number.
3. The first `INSERT` wins. The second hits the **unique index** from lesson 13 — our last line of
   defence did its job: no duplicate reached the table. But in the web app, this user would get an error
   page.
4. The invoice created by the demo has no lines. Clean up afterwards with
   `php artisan tinker` → `App\Models\Invoice::doesntHave('lines')->delete();`.

Save this output: it is your "before" evidence for the advanced task.

**Commit:** `chore(invoices): add race demo command`

---

## Step 6 — The `invoice_sequences` table

```bash
php artisan make:migration create_invoice_sequences_table
```

```php
<?php

// database/migrations/xxxx_xx_xx_xxxxxx_create_invoice_sequences_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('invoice_sequences', function (Blueprint $table) {
            $table->integer('year')->primary();
            $table->integer('last_number');
        });

        // Continue from the invoices that already exist, so the next number is max + 1.
        DB::statement(<<<'SQL'
            INSERT INTO invoice_sequences (year, last_number)
            SELECT CAST(SUBSTR(number, 1, 4) AS INTEGER), MAX(CAST(SUBSTR(number, 6, 4) AS INTEGER))
            FROM invoices
            GROUP BY CAST(SUBSTR(number, 1, 4) AS INTEGER)
            SQL);
    }

    public function down(): void
    {
        Schema::dropIfExists('invoice_sequences');
    }
};
```

**What happens:**

1. `year` is the **primary key**: at most one row per year, and a unique index for free — which
   `ON CONFLICT` needs.
2. The `INSERT … SELECT` **backfills** the table from existing invoices: for `2026-0001` … `2026-0007`
   it inserts `(2026, 7)`. Without it, the next invoice would be `2026-0001` again and crash on the
   unique constraint. A migration that changes how data is interpreted must also migrate the data.
3. `SUBSTR` and `CAST(… AS INTEGER)` work in both PostgreSQL and SQLite, so the migration also runs in
   the SQLite test database.
4. `<<<'SQL'` is a nowdoc (PHP's multi-line string without variable interpolation).

**Check it works:** `php artisan migrate`, then
`docker compose exec postgres psql -U billable -d billable -c "select * from invoice_sequences"` shows
one row per year with the highest number you have used.

---

## Step 7 — Lock the row: the new `InvoiceNumberGenerator`

Replace the whole class:

```php
<?php

// app/Invoicing/InvoiceNumberGenerator.php

namespace App\Invoicing;

use Illuminate\Support\Facades\DB;
use InvalidArgumentException;
use LogicException;

/**
 * Gives out invoice numbers in the form YYYY-NNNN, sequential per year, with no gaps and no
 * duplicates (R11) — also when two invoices are generated at the same moment.
 *
 * The year's row in invoice_sequences is locked (SELECT … FOR UPDATE) until the surrounding
 * transaction commits, so a second caller waits and then reads the updated value.
 */
final class InvoiceNumberGenerator
{
    public function next(int $year): string
    {
        if (DB::transactionLevel() === 0) {
            throw new LogicException('InvoiceNumberGenerator::next() must run inside a database transaction.');
        }

        // 1. Make sure the row exists. ON CONFLICT DO NOTHING: safe when two callers do it at once.
        DB::table('invoice_sequences')->insertOrIgnore(['year' => $year, 'last_number' => 0]);

        // 2. Lock the row. A second caller blocks here until our transaction ends.
        $last = (int) DB::table('invoice_sequences')
            ->where('year', $year)
            ->lockForUpdate()
            ->value('last_number');

        // 3. Take the next number.
        $next = $last + 1;
        DB::table('invoice_sequences')->where('year', $year)->update(['last_number' => $next]);

        return self::format($year, $next);
    }

    public static function format(int $year, int $sequence): string
    {
        if ($sequence < 1 || $sequence > 9999) {
            throw new InvalidArgumentException("Invoice sequence must be between 1 and 9999, got {$sequence}.");
        }

        return sprintf('%04d-%04d', $year, $sequence);
    }

    public static function sequenceOf(string $number): int
    {
        if (preg_match('/^\d{4}-(\d{4})$/', $number, $matches) !== 1) {
            throw new InvalidArgumentException("\"{$number}\" is not an invoice number.");
        }

        return (int) $matches[1];
    }
}
```

**What happens:**

1. The guard refuses to run outside a transaction. Without a transaction, the lock would be released
   immediately after the `SELECT` and protect nothing. `DB::transaction()` in `InvoiceGenerator` (and in
   the demo) provides it. This turns a silent bug into a clear error.
2. `insertOrIgnore()` compiles to `INSERT … ON CONFLICT DO NOTHING` on PostgreSQL (and `INSERT OR
   IGNORE` on SQLite). On 1 January the first caller creates the row; a simultaneous second caller waits
   for the first one's transaction and then inserts nothing. A hand-written "select, and insert if not
   found" would **not** be safe: two callers could both see "no row" (README, concept 6). Laravel's
   `firstOrCreate()` survives that race since Laravel 10 (after an empty select it calls
   `createOrFirst()`, see the README), but it needs a savepoint and an exception to do it; `insertOrIgnore()` is one statement.
3. `lockForUpdate()` adds `FOR UPDATE` to the `SELECT`. PostgreSQL locks the row until `COMMIT` or
   `ROLLBACK`. Anyone else asking for the same lock **waits**. In READ COMMITTED, when the waiting
   transaction continues, it reads the **new** committed value.
4. `update()` stores the new counter in the same transaction. If saving the invoice fails later, the
   rollback also restores `last_number` — no gap.
5. `format()` and `sequenceOf()` are unchanged, so the unit tests from lesson 13 still pass.
   `sequenceOf()` is no longer used by `next()`; keep it — it is handy for the backfill checks and the
   tests.
6. On SQLite `lockForUpdate()` is silently ignored. The feature tests still pass (one process, no
   concurrency), which is exactly why **concurrency** must be checked on PostgreSQL.

> **One statement instead of three.** PostgreSQL can create, increment and lock the row in a single
> atomic statement — `ON CONFLICT DO UPDATE` locks the row it updates until commit, just like
> `FOR UPDATE`:
>
> ```php
> $next = DB::selectOne(<<<'SQL'
>     INSERT INTO invoice_sequences (year, last_number) VALUES (?, 1)
>     ON CONFLICT (year) DO UPDATE SET last_number = invoice_sequences.last_number + 1
>     RETURNING last_number
>     SQL, [$year])->last_number;
> ```
>
> It is shorter and needs one round trip instead of three; SQLite 3.35+ understands it too. We use the
> three-step version in class because each step shows one idea (make sure the row exists, lock it,
> update it) — the guarantees are the same.

### Also lock the entries

The same race exists for the entries: two requests generating an invoice for the **same** client could
both load the same un-invoiced entries. Add one line to the query in `InvoiceGenerator`:

```php
// app/Invoicing/InvoiceGenerator.php (in invoiceableEntries(), before ->get())
            ->orderBy('started_at')
            ->lockForUpdate()
            ->get();
```

Now the second request waits at this query until the first commits. Then PostgreSQL re-checks
`invoice_id IS NULL` on the updated rows, finds nothing, and the second user gets the friendly
`NothingToInvoiceException` instead of a double invoice. (The eager-loaded `projects` query is a
separate query and is not locked — only the entries need to be.)

**Check it works:** `php artisan test` — everything green, including lesson 13's tests.

**Commit:** `fix(invoices): gap-free numbering with a locked sequence row`

---

## Step 8 — Run the race again

```bash
php artisan invoices:race-demo & php artisan invoices:race-demo & wait
```

**Expected output now:**

```
[4388] asking for a number…
[4389] asking for a number…
[4388] got 2026-0004, sleeping 3 s
[4388] saved 2026-0004
[4389] got 2026-0005, sleeping 3 s
[4389] saved 2026-0005
```

**What happens:**

1. Process 4388 locks the 2026 row first and gets `0004`.
2. Process 4389 asks for the same lock and **waits** — notice there is no "got …" line for it during
   the first process's three seconds.
3. 4388 commits; the lock is released; 4389 reads `last_number = 4` and gets `0005`. The whole thing
   takes about six seconds instead of three: the two transactions were **serialised** on that one row.
   In a real request the lock is held for milliseconds, so nobody notices.

This is your "after" evidence. Clean up the demo invoices as in Step 5.

---

## Step 9 — Test the new numbering

A feature test proves the sequence behaviour that a single process can check: continuing, and the year
boundary. (The **concurrent** behaviour is what Step 8 demonstrated.)

```php
<?php

// tests/Feature/Invoicing/InvoiceNumberGeneratorTest.php

namespace Tests\Feature\Invoicing;

use App\Invoicing\InvoiceNumberGenerator;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\DB;
use LogicException;
use Tests\TestCase;

final class InvoiceNumberGeneratorTest extends TestCase
{
    use RefreshDatabase;

    public function test_numbers_are_sequential_per_year_and_restart_in_a_new_year(): void
    {
        $numbers = $this->app->make(InvoiceNumberGenerator::class);

        $taken = DB::transaction(fn (): array => [
            $numbers->next(2026),
            $numbers->next(2026),
            $numbers->next(2027),
            $numbers->next(2026),
        ]);

        $this->assertSame(['2026-0001', '2026-0002', '2027-0001', '2026-0003'], $taken);
        $this->assertDatabaseHas('invoice_sequences', ['year' => 2026, 'last_number' => 3]);
        $this->assertDatabaseHas('invoice_sequences', ['year' => 2027, 'last_number' => 1]);
    }

    public function test_it_continues_from_the_stored_last_number(): void
    {
        DB::table('invoice_sequences')->insert(['year' => 2026, 'last_number' => 41]);

        $number = DB::transaction(fn (): string => $this->app->make(InvoiceNumberGenerator::class)->next(2026));

        $this->assertSame('2026-0042', $number);
    }

    public function test_a_rolled_back_transaction_does_not_use_up_a_number(): void
    {
        $numbers = $this->app->make(InvoiceNumberGenerator::class);

        try {
            DB::transaction(function () use ($numbers): void {
                $numbers->next(2026);
                throw new \RuntimeException('Saving the invoice failed.');
            });
        } catch (\RuntimeException) {
            // expected
        }

        $this->assertSame('2026-0001', DB::transaction(fn (): string => $numbers->next(2026)));
    }

    public function test_it_refuses_to_run_outside_a_transaction(): void
    {
        DB::rollBack(); // leave the transaction that RefreshDatabase opened for this test
        $this->assertSame(0, DB::transactionLevel());

        $this->expectException(LogicException::class);

        $this->app->make(InvoiceNumberGenerator::class)->next(2026);
    }
}
```

The file name matches the unit test from lesson 13, but it is in `tests/Feature/Invoicing`, so there
is no clash.

**What happens:**

1. The first test checks sequence, year separation and the stored counter.
2. The second simulates a table that was backfilled by the migration.
3. The third proves "no gap": the number taken inside a rolled-back transaction is given out again.
4. The fourth proves the guard. `RefreshDatabase` wraps **every** test in a transaction, so
   `DB::transactionLevel()` is normally 1. This test first leaves that transaction with `DB::rollBack()`
   (nothing was written yet, so nothing is lost) and checks the level really is 0. At the end of the
   test, `RefreshDatabase` tries to roll back again; with no open transaction Laravel simply does
   nothing.
5. Because of that wrapping transaction, the first three tests would pass even without their own
   `DB::transaction()`. We keep it anyway: it states the contract ("call me inside a transaction"), and
   in the third test it is what creates the savepoint that is rolled back.

**Check it works:** `php artisan test` and `php artisan test --configuration=phpunit.pgsql.xml` (if
you did Step 4) — both green. `./vendor/bin/pint --test` — clean.

**Commit:** `test(invoices): numbering with invoice_sequences`

> **Retry on conflict.** `DB::transaction($callback, attempts: 3)` re-runs the closure when the database
> reports a deadlock or a serialization failure. Our design avoids both (one lock, always taken in the same
> order), so we do not need it — but know it exists, and only use it for transactions without side
> effects outside the database.

---

## Troubleshooting

| Error / symptom | Cause | Fix |
|---|---|---|
| `SQLSTATE[08006] … database "billable_test" does not exist` | Test database not created | `docker compose exec postgres createdb -U billable billable_test` |
| The init script in `docker-entrypoint-initdb.d` never ran | The volume already had data | Use `createdb`, or `docker compose down -v` (deletes dev data) |
| `InvoiceNumberGenerator::next() must run inside a database transaction` | Called without `DB::transaction()` | Call it only inside the generator's transaction |
| `FOR UPDATE is not allowed with aggregate functions` | You tried `max('number')` with `lockForUpdate()` | Lock the `invoice_sequences` row instead (Step 7) |
| Race demo: both processes succeed even with the old code | They did not overlap (started too far apart), or ran against SQLite | Start both in one command with `&`; check `.env` uses `pgsql` |
| Race demo: second process hangs forever | The first process is stopped at a breakpoint or Tinker holds an open transaction | Finish or close the other session; locks are released at commit/rollback |
| `Illuminate\Database\UniqueConstraintViolationException … invoice_sequences_pkey` | Two plain `insert()` calls for a new year | Use `insertOrIgnore()` |
| `Expected 6 database queries on the [sqlite] connection. 7 occurred.` — the same number with 5 and 30 entries | Your time sheet needs a different (but fixed) number of queries | Set `QUERIES` to that number (Step 3) |
| `Expected 6 database queries … ` with a **different** number for 5 and 30 entries | N+1: a query runs once per entry | Eager-load the relation the page uses (lesson 06) |
| `Call to undefined method App\Models\TimeEntry::factory()` | `TimeEntryFactory` / `HasFactory` from lesson 13 Step 11 missing | Add them as in lesson 13 |
| Query-count test: more queries than expected, all `select * from "sessions"` or `cache` | Session/cache driver is `database` in tests | `phpunit.xml` should have `SESSION_DRIVER=array` and `CACHE_STORE=array` (Laravel's default) |
| `assertSessionHasErrors` fails, but the page shows the error | The controller used `->with('error', …)`, not `withErrors()` | Use `assertSessionHas('error')` for flash messages |
| Next invoice after the migration is `2026-0001` and crashes | Backfill missing in the migration | Add the `INSERT … SELECT` from Step 6 (new migration if the old one already ran) |

---

## Recap

- **Feature tests** over HTTP: `ClientCrudTest`, `InvoiceFlowTest` (numbers, totals, links, overlap, R13).
- **Query-count test** for the time sheet with `expectsDatabaseQueryCount()` and `#[TestWith]` — the count must not depend on the data size.
- **Optional PostgreSQL test database** with `phpunit.pgsql.xml`; SQLite stays the fast default.
- **Race demo** command: saw the duplicate-number race, then saw it disappear.
- **`invoice_sequences`** migration with backfill; `InvoiceNumberGenerator` uses `insertOrIgnore()` + `lockForUpdate()` inside the transaction, with a guard.
- **Entries locked** while being invoiced, so the same hours are never billed twice.
- The unique constraint on `invoices.number` stays — the last line of defence.

---

## Independent work — solutions

<details>
<summary>Task 1 — More feature tests for the main flows (Basic)</summary>

Three examples. Adapt the field names to your forms.

```php
<?php

// tests/Feature/MainFlowsTest.php

namespace Tests\Feature;

use App\Enums\InvoiceStatus;
use App\Models\Invoice;
use App\Models\Project;
use App\Models\TimeEntry;
use Carbon\CarbonImmutable;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class MainFlowsTest extends TestCase
{
    use RefreshDatabase;

    protected function setUp(): void
    {
        parent::setUp();
        $this->travelTo(CarbonImmutable::parse('2026-09-30 12:00'));
    }

    public function test_a_logged_entry_appears_on_the_time_sheet(): void
    {
        $project = Project::factory()->create(['name' => 'Website redesign']);

        $this->post(route('time-entries.store'), [
            'project_id' => $project->id,
            'started_at' => '2026-09-30 09:00',
            'ended_at' => '2026-09-30 10:30',
            'description' => 'Homepage layout',
            'billable' => '1',
        ])->assertRedirect(route('time-entries.index'));

        $this->assertDatabaseHas('time_entries', ['project_id' => $project->id, 'description' => 'Homepage layout']);
        $this->get(route('time-entries.index'))->assertOk()->assertSee('Homepage layout');
    }

    public function test_a_sent_invoice_can_be_marked_paid_but_a_draft_cannot(): void
    {
        $invoice = Invoice::factory()->create(); // a draft

        $this->post(route('invoices.mark-paid', $invoice))->assertSessionHas('error');
        $this->assertSame(InvoiceStatus::Draft, $invoice->fresh()->status);

        $this->post(route('invoices.send', $invoice));
        $this->post(route('invoices.mark-paid', $invoice))->assertSessionHas('status');
        $this->assertSame(InvoiceStatus::Paid, $invoice->fresh()->status);
    }

    public function test_an_invoiced_entry_cannot_be_deleted(): void
    {
        $project = Project::factory()->hourly(6000)->create();
        $entry = TimeEntry::factory()->for($project)->between('2026-09-29 09:00', '2026-09-29 10:00')->create();
        $this->post(route('invoices.store'), ['client_id' => $project->client_id, 'up_to' => '2026-09-30']);

        $this->delete(route('time-entries.destroy', $entry))->assertSessionHas('error');

        $this->assertModelExists($entry);
    }
}
```

The factories from lesson 13 keep each test to its point: `Invoice::factory()->create()` is "some draft
invoice", `->sent()` or `->paid()` would give the other states. The last test still generates its
invoice through HTTP, because "the entry is linked to an invoice" is what it is about.

The last test needs the `destroy` route for time entries (`Route::resource('time-entries', …)` with
`destroy` included), `TimeEntryService::delete()` with the R14 guard, and the try/catch in
`TimeEntryController::destroy()` — all from lesson 13, Step 8 (the box *"If you do not have delete yet"*
if you skipped lesson 07 step 9).
</details>

<details>
<summary>Task 2 — Indexes justified by EXPLAIN (Basic)</summary>

**1. Realistic data.** A seeder with many entries:

```php
<?php

// database/seeders/PerformanceSeeder.php

namespace Database\Seeders;

use App\Models\Project;
use Carbon\CarbonImmutable;
use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\DB;

class PerformanceSeeder extends Seeder
{
    public function run(): void
    {
        $projectIds = Project::query()->pluck('id')->all();
        $start = CarbonImmutable::parse('2025-01-01 08:00');
        $rows = [];

        for ($i = 0; $i < 10_000; $i++) {
            $startedAt = $start->addMinutes($i * 47);
            $rows[] = [
                'project_id' => $projectIds[$i % count($projectIds)],
                'started_at' => $startedAt,
                'ended_at' => $startedAt->addMinutes(45),
                'description' => "Seeded entry {$i}",
                'billable' => true,
                'created_at' => now(),
                'updated_at' => now(),
            ];
        }

        foreach (array_chunk($rows, 1000) as $chunk) {
            DB::table('time_entries')->insert($chunk);
        }
    }
}
```

Run `php artisan db:seed --class=PerformanceSeeder`. Rows are inserted with the query builder in
chunks of 1,000 — 10 inserts instead of 10,000, and no model events.

**2. Plans before.** In `psql` (`docker compose exec postgres psql -U billable -d billable`), run
`ANALYZE;` and then:

```sql
EXPLAIN ANALYZE SELECT * FROM time_entries WHERE invoice_id = 1;
EXPLAIN ANALYZE SELECT project_id, SUM(amount_cents) FROM invoice_lines WHERE project_id IN (1, 2) GROUP BY project_id;
EXPLAIN ANALYZE SELECT * FROM invoices WHERE client_id = 1;
```

Expect `Seq Scan on time_entries … Filter: (invoice_id = 1) … Rows Removed by Filter: 10000` or similar.
(`invoice_lines` and `invoices` are small in development, so they may show `Seq Scan` even with an
index — say so in your notes, and seed more invoices if you want to see the switch.)

**3. Migration.** `php artisan make:migration add_foreign_key_indexes`:

```php
<?php

// database/migrations/xxxx_xx_xx_xxxxxx_add_foreign_key_indexes.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('time_entries', fn (Blueprint $table) => $table->index('invoice_id'));
        Schema::table('invoice_lines', fn (Blueprint $table) => $table->index('project_id'));
        Schema::table('invoices', fn (Blueprint $table) => $table->index('client_id'));
    }

    public function down(): void
    {
        Schema::table('invoices', fn (Blueprint $table) => $table->dropIndex(['client_id']));
        Schema::table('invoice_lines', fn (Blueprint $table) => $table->dropIndex(['project_id']));
        Schema::table('time_entries', fn (Blueprint $table) => $table->dropIndex(['invoice_id']));
    }
};
```

`invoice_lines.invoice_id` is used by `$invoice->lines` on every invoice page — add it too if your
EXPLAIN shows a `Seq Scan` there. Do **not** add `projects.client_id`: lesson 03 already created
`projects_client_id_index`.

**4. Plans after.** Run the same `EXPLAIN ANALYZE`; the first now shows
`Index Scan using time_entries_invoice_id_index on time_entries`. Put before/after and one sentence per
index into `docs/performance.md`, e.g. *"`time_entries.invoice_id`: used when a draft is deleted and when
the invoice page lists its entries; PostgreSQL does not index foreign keys automatically."*
</details>

<details>
<summary>Task 3 — Query count for the invoices page (Intermediate)</summary>

```php
<?php

// tests/Feature/InvoiceIndexQueryCountTest.php

namespace Tests\Feature;

use App\Enums\InvoiceStatus;
use App\Models\Invoice;
use Illuminate\Foundation\Testing\RefreshDatabase;
use PHPUnit\Framework\Attributes\TestWith;
use Tests\TestCase;

final class InvoiceIndexQueryCountTest extends TestCase
{
    use RefreshDatabase;

    #[TestWith([30])]
    #[TestWith([60])]
    public function test_the_invoices_page_needs_the_same_few_queries_for_any_number_of_invoices(int $count): void
    {
        Invoice::factory()
            ->count($count)
            ->sequence(
                ['status' => InvoiceStatus::Draft],
                ['status' => InvoiceStatus::Sent],
                ['status' => InvoiceStatus::Paid],
            )
            ->create();

        $this->expectsDatabaseQueryCount(3);

        $this->get(route('invoices.index'))->assertOk();
    }
}
```

`Invoice::factory()->count(30)` creates 30 invoices, each for its **own** client (the factory's
`'client_id' => Client::factory()`), which is exactly what an N+1 test needs. `sequence()` mixes the
three statuses, so the page shows every badge. With `with('client')->paginate(20)` the page runs 3
queries (count, invoices, clients) for 30 and for 60 invoices. Remove `with('client')`: `preventLazyLoading()` throws, or — if you switched that off —
the count jumps to 22.
</details>

<details>
<summary>Task 4 — A security fix with a test (Intermediate)</summary>

Example: a view printed the client name with `{!! $client->name !!}` "because it has a special
character". That is stored XSS. The test first:

```php
<?php

// tests/Feature/ClientXssTest.php

namespace Tests\Feature;

use App\Models\Client;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class ClientXssTest extends TestCase
{
    use RefreshDatabase;

    public function test_client_names_are_escaped_on_the_detail_page(): void
    {
        $client = Client::factory()->create(['name' => '<script>alert("x")</script>']);

        $this->get(route('clients.show', $client))
            ->assertOk()
            ->assertDontSee('<script>alert("x")</script>', escape: false)
            ->assertSee('&lt;script&gt;', escape: false);
    }
}
```

`assertSee(..., escape: false)` searches the raw HTML. The test fails while the view uses `{!! !!}`;
changing it to `{{ $client->name }}` makes it pass. Commit as
`fix(security): escape client name on detail page (stored XSS)`.

Another common find: `TimeEntry::create($request->all())`. Test it by posting an extra
`'invoice_id' => 1` and asserting the saved entry has `invoice_id` `null`; fix it with
`$request->validated()`.
</details>

<details>
<summary>Task 5 — Concurrency evidence (Advanced)</summary>

1. Make the demo repeatable: add `{--times=1}` to the signature and loop the transaction that many times.
2. **Before:** `git stash` (or check out the lesson 13 commit of `InvoiceNumberGenerator.php` only:
   `git checkout <commit> -- app/Invoicing/InvoiceNumberGenerator.php`), then run
   `php artisan invoices:race-demo --sleep=1 --times=5 & php artisan invoices:race-demo --sleep=1 --times=5 & wait`.
   Copy the output with the `UniqueConstraintViolationException`.
3. **After:** restore the new class (`git checkout HEAD -- app/Invoicing/InvoiceNumberGenerator.php`) and
   run the same command. Copy the output: ten "saved" lines, ten different, consecutive numbers.
4. Check in the database: `SELECT number FROM invoices ORDER BY number;` has no gaps.
5. In `docs/algorithm.md`, add both outputs and two sentences, e.g.: *"With max + 1 both processes read
   the same maximum under READ COMMITTED and the unique constraint rejected the second insert. With the
   locked sequence row the second process waited for the first commit and continued from the new value;
   the run took twice as long because the transactions were serialised."*
</details>
