# 13 — A complex component: invoice generation · Laravel

[← Concepts](README.md) · Starting point: end of lesson 12 · Estimated time in class: 2 h

## What we build today

- Three migrations: `invoices`, `invoice_lines`, and `time_entries.invoice_id`
- `App\Enums\InvoiceStatus` with `canEdit()` and `canTransitionTo()`
- Models `Invoice` and `InvoiceLine`, plus new relations on `Client` and `TimeEntry`
- `App\Invoicing\TimeSlot` and `OverlapDetector` (pure, sort + sweep) with a data-provider unit test
- `App\Invoicing\InvoiceNumberGenerator` (max + 1, formatted `YYYY-NNNN`) with unit tests
- `App\Invoicing\InvoiceGenerator` — the orchestrator, all in one `DB::transaction`
- Domain exceptions that explain *why* generation failed
- `App\Services\InvoiceService` for send / mark paid / delete draft, and the R14 guard in `TimeEntryService`
- `InvoiceController` with list, create form, show page and actions
- `TimeEntryFactory` and `InvoiceFactory` (with states), and a feature test for the whole generator with
  `RefreshDatabase`, factories and `travelTo()`

**Final result:** on `/invoices/create` you pick a client and a date and press *Generate*. You land on
the invoice page: number `2026-0001`, one line per project such as `Website redesign — 2 h 15 min`,
subtotal, VAT 24 %, total, due date, status *Draft*, and buttons *Send* and *Delete*. If two entries
overlap you stay on the form and see which two.

### What this lesson uses from earlier lessons

This walkthrough calls code you wrote in lessons 06–12. If your names differ, adapt the calls — do not
rename your classes just for this lesson.

| Used here | Expected shape (from earlier lessons) |
|---|---|
| `App\Support\Money` (lessons 10, 11) | `Money::fromCents(int)`, public `int $cents`, `add()`, `subtract()`, `percentageBp(int)`, `isZero()`, `format()`, and `Money::sum(Money ...$amounts)` (lesson 11) |
| `App\Support\TimeRounding` (lesson 10) | `TimeRounding::roundUp(int $minutes, int $increment): int` (static, pure) |
| `App\Support\DueDateCalculator` (lesson 10) | `dueDate(CarbonImmutable $issuedOn): CarbonImmutable` (R12) |
| `config/billable.php` (lesson 10) | `'vat_rate_bp' => 2400` |
| `App\Enums\BillingType` (lessons 03, 08) | cases `Hourly`, `Fixed`, `Capped`, and `Internal` if you did the lesson 08 basic task (Step 7 says what to change if you did not) |
| `App\Billing\BillingStrategyResolver` (lessons 08, 10) | `forProject(Project $project): BillingStrategy`; strategy `calculate(Project, int $billableMinutes, Money $alreadyBilled): Money` |
| `App\Models\TimeEntry` (lessons 06, 10) | `durationMinutes(): int`, `project()`; fillable `project_id`, `started_at`, `ended_at`, `description`, `billable` |
| `App\Services\TimeEntryService` (lesson 07) | `log(array $data)`; `update(TimeEntry $entry, array $data)` and `delete(TimeEntry $entry)` only if you did lesson 07 step 9 — otherwise Step 8 shows how to add `delete()` |
| Factories (lesson 03) | `ClientFactory`, `ProjectFactory` |
| Relations | `Client::projects()`, `Project::client()`, `Project::timeEntries()`, `TimeEntry::project()` |

---

## Step 1 — Migrations

We need two new tables and one new column. The order matters: `invoice_lines` and `time_entries`
point **to** `invoices`, so `invoices` must exist first. Laravel runs migrations in file-name order, and
the file names start with a timestamp, so create them in this order:

```bash
php artisan make:migration create_invoices_table
php artisan make:migration create_invoice_lines_table
php artisan make:migration add_invoice_id_to_time_entries_table
```

```php
<?php

// database/migrations/xxxx_xx_xx_xxxxxx_create_invoices_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('invoices', function (Blueprint $table) {
            $table->id();
            $table->foreignId('client_id')->constrained()->restrictOnDelete();
            $table->string('number', 9)->unique();
            $table->string('status', 10)->default('draft');
            $table->date('issued_on');
            $table->date('due_on');
            $table->integer('vat_rate_bp');
            $table->bigInteger('subtotal_cents');
            $table->bigInteger('vat_cents');
            $table->bigInteger('total_cents');
            $table->timestamp('created_at');
            $table->timestamp('updated_at');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('invoices');
    }
};
```

```php
<?php

// database/migrations/xxxx_xx_xx_xxxxxx_create_invoice_lines_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('invoice_lines', function (Blueprint $table) {
            $table->id();
            $table->foreignId('invoice_id')->constrained()->cascadeOnDelete();
            $table->foreignId('project_id')->constrained()->restrictOnDelete();
            $table->string('description');
            $table->integer('minutes');
            $table->bigInteger('amount_cents');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('invoice_lines');
    }
};
```

```php
<?php

// database/migrations/xxxx_xx_xx_xxxxxx_add_invoice_id_to_time_entries_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('time_entries', function (Blueprint $table) {
            $table->foreignId('invoice_id')->nullable()->constrained()->restrictOnDelete();
        });
    }

    public function down(): void
    {
        Schema::table('time_entries', function (Blueprint $table) {
            $table->dropConstrainedForeignId('invoice_id');
        });
    }
};
```

**What happens:**

1. `string('number', 9)->unique()` creates `varchar(9)` with a unique index. `2026-0001` is exactly 9
   characters. The unique index is our last line of defence against duplicate numbers (R11).
2. `status` is plain text with the default `draft`. The enum in Step 2 decides which values are valid.
3. All money columns are `bigint` cents (R1), like the money columns on `projects`. `vat_rate_bp`
   stores 24 % as `2400` (R2); it is a rate, not money, so `integer` is enough.
4. `invoice_lines.invoice_id` uses `cascadeOnDelete()`: when a draft invoice is deleted, PostgreSQL
   deletes its lines too. Lines have no meaning without their invoice.
5. `time_entries.invoice_id` is `nullable()` (an entry is not invoiced yet) and uses
   `restrictOnDelete()`. That is deliberate: the database **refuses** to delete an invoice that still has
   entries. Our code must release the entries first (Step 8). If someone forgets, they get an error
   instead of silently losing the link.
6. `invoices` gets the two `NOT NULL` timestamp columns, as in lesson 03. `invoice_lines` has no
   `created_at` / `updated_at` — PROJECT.md does not list them, and a line never changes on its own.
7. `dropConstrainedForeignId()` in `down()` drops the foreign key and the column together.

**Check it works:**

```bash
php artisan migrate
php artisan migrate:status
```

The three new migrations are listed as `Ran`. Then run `php artisan migrate:rollback --step=3` and
`php artisan migrate` again — both must work without errors. A migration you cannot roll back is a bug.

**Commit:** `feat(invoices): add invoices, invoice_lines and time_entries.invoice_id`

---

## Step 2 — The `InvoiceStatus` enum

The status is a small state machine (README, concept 6). We put all its rules in one enum.

```php
<?php

// app/Enums/InvoiceStatus.php

namespace App\Enums;

enum InvoiceStatus: string
{
    case Draft = 'draft';
    case Sent = 'sent';
    case Paid = 'paid';

    public function label(): string
    {
        return match ($this) {
            self::Draft => 'Draft',
            self::Sent => 'Sent',
            self::Paid => 'Paid',
        };
    }

    /**
     * Only a draft may be edited or deleted (R13).
     */
    public function canEdit(): bool
    {
        return $this === self::Draft;
    }

    /**
     * The allowed moves: draft → sent → paid. Everything else is forbidden.
     */
    public function canTransitionTo(self $to): bool
    {
        return match ($this) {
            self::Draft => $to === self::Sent,
            self::Sent => $to === self::Paid,
            self::Paid => false,
        };
    }
}
```

**What happens:**

1. A **backed enum** maps each case to the lowercase string stored in the database (`'draft'`).
2. `match` must cover every case. If you add `case Cancelled` later and forget to handle it, PHP throws
   `UnhandledMatchError` the first time it is used — much better than a silent wrong answer.
3. `canTransitionTo()` is the transition table from the README written as code. Staying in the same
   state (`Draft → Draft`) is also `false`: it is not a transition.

**Check it works:**

```bash
php artisan tinker
> App\Enums\InvoiceStatus::Draft->canTransitionTo(App\Enums\InvoiceStatus::Sent)
= true
> App\Enums\InvoiceStatus::Draft->canTransitionTo(App\Enums\InvoiceStatus::Paid)
= false
```

---

## Step 3 — Models and relations

```php
<?php

// app/Models/Invoice.php

namespace App\Models;

use App\Enums\InvoiceStatus;
use App\Support\Money;
use Database\Factories\InvoiceFactory;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Invoice extends Model
{
    /** @use HasFactory<InvoiceFactory> */
    use HasFactory;

    protected $fillable = [
        'client_id',
        'number',
        'status',
        'issued_on',
        'due_on',
        'vat_rate_bp',
        'subtotal_cents',
        'vat_cents',
        'total_cents',
    ];

    protected function casts(): array
    {
        return [
            'status' => InvoiceStatus::class,
            'issued_on' => 'immutable_date',
            'due_on' => 'immutable_date',
            'vat_rate_bp' => 'integer',
            'subtotal_cents' => 'integer',
            'vat_cents' => 'integer',
            'total_cents' => 'integer',
        ];
    }

    public function client(): BelongsTo
    {
        return $this->belongsTo(Client::class);
    }

    public function lines(): HasMany
    {
        return $this->hasMany(InvoiceLine::class)->orderBy('id');
    }

    public function timeEntries(): HasMany
    {
        return $this->hasMany(TimeEntry::class);
    }

    public function subtotal(): Money
    {
        return Money::fromCents($this->subtotal_cents);
    }

    public function vat(): Money
    {
        return Money::fromCents($this->vat_cents);
    }

    public function total(): Money
    {
        return Money::fromCents($this->total_cents);
    }
}
```

```php
<?php

// app/Models/InvoiceLine.php

namespace App\Models;

use App\Support\Money;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class InvoiceLine extends Model
{
    public $timestamps = false;

    protected $fillable = ['project_id', 'description', 'minutes', 'amount_cents'];

    protected function casts(): array
    {
        return [
            'minutes' => 'integer',
            'amount_cents' => 'integer',
        ];
    }

    public function invoice(): BelongsTo
    {
        return $this->belongsTo(Invoice::class);
    }

    public function project(): BelongsTo
    {
        return $this->belongsTo(Project::class);
    }

    public function amount(): Money
    {
        return Money::fromCents($this->amount_cents);
    }
}
```

Now add the other side of the relations. In `app/Models/Client.php`, next to `projects()`:

```php
// app/Models/Client.php (add inside the class; import HasMany if it is not imported yet)
public function invoices(): HasMany
{
    return $this->hasMany(Invoice::class);
}
```

In `app/Models/TimeEntry.php`, add `invoice_id` **nowhere** in `$fillable` (only the generator sets it,
with a query), and add these two methods next to `project()`:

```php
// app/Models/TimeEntry.php (add inside the class; import BelongsTo if needed)
public function invoice(): BelongsTo
{
    return $this->belongsTo(Invoice::class);
}

public function isInvoiced(): bool
{
    return $this->invoice_id !== null;
}
```

**What happens:**

1. The `status` cast turns the database string into an `InvoiceStatus` case and back. You write
   `$invoice->status === InvoiceStatus::Draft`, never `=== 'draft'`.
2. `immutable_date` gives a `CarbonImmutable` with the time at midnight. Immutable means methods like
   `addDays()` return a new object instead of changing this one — no accidental changes to the stored
   date.
3. The integer casts make sure the cents are PHP `int`, whatever the database driver returns.
4. `lines()` has `orderBy('id')` so the lines always appear in the order they were created.
5. `subtotal()`, `vat()` and `total()` wrap the stored cents in `Money` so views can call `->format()`.
   The model stores integers; `Money` does the arithmetic and formatting.
6. `InvoiceLine` sets `$timestamps = false` because its table has no `created_at`/`updated_at`.
   Without it, Eloquent tries to write those columns and PostgreSQL answers
   `column "updated_at" of relation "invoice_lines" does not exist`.
7. `invoice_id` is not fillable on purpose. A form must never be able to move an entry onto an invoice
   (mass assignment — lesson 14 returns to this).
8. `HasFactory` connects `Invoice` to `InvoiceFactory`, which we write in Step 11 together with the
   tests that use it.

**Commit:** `feat(invoices): add Invoice, InvoiceLine and InvoiceStatus`

---

## Step 4 — `TimeSlot` and `OverlapDetector`

The overlap algorithm does not need Eloquent at all. It needs a start, an end, and something to name
the entry in the error message. So we give it a tiny value object instead of models. That keeps it pure,
and a pure class can be tested with `PHPUnit\Framework\TestCase` — no Laravel, no database.

```php
<?php

// app/Invoicing/TimeSlot.php

namespace App\Invoicing;

use DateTimeInterface;
use InvalidArgumentException;

/**
 * The part of a time entry that the overlap check needs.
 */
final readonly class TimeSlot
{
    public function __construct(
        public int $entryId,
        public DateTimeInterface $start,
        public DateTimeInterface $end,
        public string $description,
    ) {
        if ($end <= $start) {
            throw new InvalidArgumentException("Time slot {$entryId} must end after it starts.");
        }
    }

    /**
     * Touching slots (one ends exactly when the other starts) do not overlap.
     */
    public function overlaps(self $other): bool
    {
        return $this->start < $other->end && $other->start < $this->end;
    }
}
```

```php
<?php

// app/Invoicing/OverlapDetector.php

namespace App\Invoicing;

/**
 * Finds two time slots that overlap (R10).
 *
 * Sorts the slots by start (then end) and sweeps through them once, remembering the slot with the
 * latest end seen so far. Time O(n log n) because of the sort, extra space O(n) for the sorted copy.
 */
final class OverlapDetector
{
    /**
     * @param  list<TimeSlot>  $slots  in any order
     * @return array{0: TimeSlot, 1: TimeSlot}|null the first overlapping pair found, or null
     */
    public function find(array $slots): ?array
    {
        usort($slots, fn (TimeSlot $a, TimeSlot $b): int => [$a->start, $a->end] <=> [$b->start, $b->end]);

        $latest = null;

        foreach ($slots as $slot) {
            if ($latest !== null && $slot->start < $latest->end) {
                return [$latest, $slot];
            }

            if ($latest === null || $slot->end > $latest->end) {
                $latest = $slot;
            }
        }

        return null;
    }
}
```

**What happens:**

1. `final readonly class` (PHP 8.2+) makes every promoted property read-only. A slot cannot change after
   it is created.
2. The constructor refuses a slot that ends before or when it starts. R4 already guarantees this for
   saved entries, but a value object should protect itself.
3. `DateTimeInterface` accepts both `Carbon` and `CarbonImmutable` (and plain PHP dates). PHP can compare
   date objects with `<`, `>` and `<=>`.
4. `usort()` sorts the array **in place** — but PHP passes arrays **by value** (copy on write), so it
   sorts our local copy. The caller's array keeps its order.
5. The comparator compares two-element arrays: first by start, and when the starts are equal, by end.
   The spaceship operator `<=>` returns -1, 0 or 1, which is what `usort()` needs.
6. The loop is the sweep from the README. `$latest` is the slot with the latest end so far. The first
   `if` finds an overlap (strict `<`, so touching is allowed). The second `if` moves `$latest` forward
   only when this slot ends later.
7. It returns the pair so the exception can name **both** entries.

Now the tests. A **data provider** is a static method that returns many input/expected pairs; PHPUnit
runs the test once per row and prints the row's name when it fails.

```php
<?php

// tests/Unit/Invoicing/OverlapDetectorTest.php

namespace Tests\Unit\Invoicing;

use App\Invoicing\OverlapDetector;
use App\Invoicing\TimeSlot;
use DateTimeImmutable;
use InvalidArgumentException;
use PHPUnit\Framework\Attributes\DataProvider;
use PHPUnit\Framework\TestCase;

final class OverlapDetectorTest extends TestCase
{
    /**
     * Each slot is [id, start, end]. A time without a date means 2026-09-01.
     *
     * @return array<string, array{list<array{int, string, string}>, list<int>|null}>
     */
    public static function cases(): array
    {
        return [
            'no slots' => [[], null],
            'one slot' => [[[1, '09:00', '10:00']], null],
            'separate slots' => [[[1, '09:00', '10:00'], [2, '11:00', '12:00']], null],
            'touching slots are allowed' => [[[1, '09:00', '10:00'], [2, '10:00', '11:00']], null],
            'same hours on different days' => [[[1, '09:00', '10:00'], [2, '2026-09-02 09:00', '2026-09-02 10:00']], null],
            'simple overlap' => [[[1, '09:00', '10:30'], [2, '10:00', '11:00']], [1, 2]],
            'nested slot' => [[[1, '09:00', '12:00'], [2, '10:00', '11:00']], [1, 2]],
            'same start time' => [[[1, '09:00', '10:00'], [2, '09:00', '09:30']], [2, 1]],
            'chain of touching slots, then an overlap' => [
                [[1, '09:00', '10:00'], [2, '10:00', '11:00'], [3, '11:00', '12:00'], [4, '11:30', '12:30']],
                [3, 4],
            ],
            'unsorted input' => [[[3, '13:00', '14:00'], [2, '10:30', '11:30'], [1, '10:00', '11:00']], [1, 2]],
        ];
    }

    /**
     * @param  list<array{int, string, string}>  $rows
     * @param  list<int>|null  $expectedIds
     */
    #[DataProvider('cases')]
    public function test_it_finds_the_first_overlapping_pair(array $rows, ?array $expectedIds): void
    {
        $slots = array_map(fn (array $row): TimeSlot => self::slot(...$row), $rows);

        $pair = (new OverlapDetector)->find($slots);

        $actualIds = $pair === null ? null : [$pair[0]->entryId, $pair[1]->entryId];
        $this->assertSame($expectedIds, $actualIds);
    }

    public function test_a_slot_must_end_after_it_starts(): void
    {
        $this->expectException(InvalidArgumentException::class);

        self::slot(1, '10:00', '10:00');
    }

    private static function slot(int $id, string $start, string $end): TimeSlot
    {
        return new TimeSlot($id, self::time($start), self::time($end), "Entry {$id}");
    }

    private static function time(string $value): DateTimeImmutable
    {
        return new DateTimeImmutable(str_contains($value, '-') ? $value : "2026-09-01 {$value}");
    }
}
```

**What happens:**

1. The test class extends `PHPUnit\Framework\TestCase`, not `Tests\TestCase`. Laravel is not booted, so
   the test runs in about a millisecond.
2. `#[DataProvider('cases')]` (PHPUnit 11+) links the test to the provider. The provider must be
   `public static`.
3. Each row is small and readable: `[1, '09:00', '10:00']`. The helper turns it into a `TimeSlot`.
4. `'same start time'` expects `[2, 1]`: after sorting by (start, end), slot 2 (ends 09:30) comes
   before slot 1 (ends 10:00), so slot 2 is `$latest` when slot 1 is checked.
5. `'chain of touching slots, then an overlap'` proves that touching (1→2→3) is never reported, but the
   real overlap at the end is.
6. `'unsorted input'` proves the detector does not rely on the caller sorting.

**Check it works:**

```bash
php artisan test --filter=OverlapDetectorTest
```

Expected: `Tests: 11 passed`. Now break the code on purpose: change `<` to `<=` in `find()` and run the
test again. The row `touching slots are allowed` fails. Change it back. A test you have never seen fail
is a test you cannot trust.

**Commit:** `feat(invoices): add OverlapDetector with sort-and-sweep algorithm`

---

## Step 5 — Exceptions that explain why

All invoicing errors share one parent class, so the controller can catch "any invoicing problem" with one
`catch`, and nothing else.

```php
<?php

// app/Invoicing/InvoicingException.php

namespace App\Invoicing;

use DomainException;

/**
 * A business rule stopped an invoicing action. The message is safe to show to the user.
 */
abstract class InvoicingException extends DomainException {}
```

```php
<?php

// app/Invoicing/NothingToInvoiceException.php

namespace App\Invoicing;

use App\Models\Client;
use DateTimeInterface;

final class NothingToInvoiceException extends InvoicingException
{
    public static function for(Client $client, DateTimeInterface $upTo): self
    {
        return new self(sprintf(
            '%s has no billable, un-invoiced time entries up to %s.',
            $client->name,
            $upTo->format('d.m.Y'),
        ));
    }
}
```

```php
<?php

// app/Invoicing/OverlappingEntriesException.php

namespace App\Invoicing;

final class OverlappingEntriesException extends InvoicingException
{
    public function __construct(
        public readonly TimeSlot $first,
        public readonly TimeSlot $second,
    ) {
        parent::__construct(sprintf(
            'Cannot generate the invoice: %s overlaps %s. Fix one of the entries and try again.',
            self::describe($first),
            self::describe($second),
        ));
    }

    private static function describe(TimeSlot $slot): string
    {
        return sprintf(
            'entry #%d "%s" (%s–%s)',
            $slot->entryId,
            $slot->description,
            $slot->start->format('d.m.Y H:i'),
            $slot->end->format('H:i'),
        );
    }
}
```

```php
<?php

// app/Invoicing/InvalidStatusTransitionException.php

namespace App\Invoicing;

use App\Enums\InvoiceStatus;

final class InvalidStatusTransitionException extends InvoicingException
{
    public static function between(InvoiceStatus $from, InvoiceStatus $to): self
    {
        return new self("An invoice cannot go from {$from->value} to {$to->value}.");
    }
}
```

```php
<?php

// app/Invoicing/InvoiceNotEditableException.php

namespace App\Invoicing;

use App\Models\Invoice;

final class InvoiceNotEditableException extends InvoicingException
{
    public static function for(Invoice $invoice): self
    {
        return new self("Invoice {$invoice->number} is {$invoice->status->value}. Only draft invoices can be changed or deleted.");
    }
}
```

```php
<?php

// app/Invoicing/TimeEntryLockedException.php

namespace App\Invoicing;

use App\Models\TimeEntry;

final class TimeEntryLockedException extends InvoicingException
{
    public static function for(TimeEntry $entry): self
    {
        return new self("Time entry #{$entry->id} is on an invoice and cannot be changed. Delete the draft invoice first to release it.");
    }
}
```

**What happens:**

1. `DomainException` is PHP's built-in class for "a rule of the business domain was broken". Our
   abstract `InvoicingException` groups all of ours.
2. Named constructors (`NothingToInvoiceException::for(...)`) build the message in one place, so every
   caller gets the same wording.
3. `OverlappingEntriesException` keeps both slots as public properties. The page shows the message; a
   test can also check `$e->first->entryId`.
4. The overlap message contains the entry numbers, descriptions, the date and both times — everything
   the user needs to find the entries on the time sheet (R10).

---

## Step 6 — `InvoiceNumberGenerator`

```php
<?php

// app/Invoicing/InvoiceNumberGenerator.php

namespace App\Invoicing;

use App\Models\Invoice;
use InvalidArgumentException;

/**
 * Gives out invoice numbers in the form YYYY-NNNN, sequential per year (R11).
 *
 * Must be called inside the transaction that saves the invoice. Lesson 14 replaces "max + 1" with a
 * locked sequence row, because two parallel requests can read the same maximum.
 */
final class InvoiceNumberGenerator
{
    public function next(int $year): string
    {
        $last = Invoice::query()
            ->where('number', 'like', $year.'-%')
            ->max('number');

        $sequence = $last === null ? 1 : self::sequenceOf($last) + 1;

        return self::format($year, $sequence);
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

1. `where('number', 'like', '2026-%')` keeps only this year's invoices. `max('number')` runs
   `SELECT MAX(number) ...` in the database and returns `null` when there are none.
2. Because every number is zero-padded to four digits, the text maximum is also the numeric maximum.
3. `sequenceOf('2026-0041')` checks the format with a regular expression and returns `41`. If someone
   ever puts a strange value into the column, we fail loudly instead of producing `2026-0001` again.
4. `format(2026, 42)` returns `2026-0042`. `%04d` means "integer, at least 4 digits, pad with zeros".
5. `format()` refuses 10,000: `2026-10000` would not fit in `varchar(9)`. A clear error is better than
   a database truncation error.
6. `format()` and `sequenceOf()` are `static` and pure, so they get pure unit tests. `next()` needs the
   database; the feature test in Step 11 covers it.

```php
<?php

// tests/Unit/Invoicing/InvoiceNumberGeneratorTest.php

namespace Tests\Unit\Invoicing;

use App\Invoicing\InvoiceNumberGenerator;
use InvalidArgumentException;
use PHPUnit\Framework\Attributes\DataProvider;
use PHPUnit\Framework\TestCase;

final class InvoiceNumberGeneratorTest extends TestCase
{
    /** @return array<string, array{int, int, string}> */
    public static function validNumbers(): array
    {
        return [
            'first of the year' => [2026, 1, '2026-0001'],
            'two digits' => [2026, 42, '2026-0042'],
            'last possible' => [2026, 9999, '2026-9999'],
            'another year' => [2027, 1, '2027-0001'],
        ];
    }

    #[DataProvider('validNumbers')]
    public function test_it_formats_and_parses_numbers(int $year, int $sequence, string $number): void
    {
        $this->assertSame($number, InvoiceNumberGenerator::format($year, $sequence));
        $this->assertSame($sequence, InvoiceNumberGenerator::sequenceOf($number));
    }

    /** @return array<string, array{int}> */
    public static function invalidSequences(): array
    {
        return ['zero' => [0], 'negative' => [-1], 'too large' => [10000]];
    }

    #[DataProvider('invalidSequences')]
    public function test_it_refuses_sequences_outside_the_format(int $sequence): void
    {
        $this->expectException(InvalidArgumentException::class);

        InvoiceNumberGenerator::format(2026, $sequence);
    }

    /** @return array<string, array{string}> */
    public static function invalidNumbers(): array
    {
        return ['no dash' => ['20260001'], 'short sequence' => ['2026-01'], 'prefix' => ['INV-2026-0001'], 'empty' => ['']];
    }

    #[DataProvider('invalidNumbers')]
    public function test_it_refuses_to_parse_malformed_numbers(string $number): void
    {
        $this->expectException(InvalidArgumentException::class);

        InvoiceNumberGenerator::sequenceOf($number);
    }
}
```

**Check it works:** `php artisan test --filter=InvoiceNumberGeneratorTest` → `Tests: 11 passed`.

**Commit:** `feat(invoices): add InvoiceNumberGenerator`

---

## Step 7 — `InvoiceGenerator`, the orchestrator

This is the centre of the lesson. Read it top to bottom once before typing: `generate()` should read
like the list in the README's "Today's goal".

```php
<?php

// app/Invoicing/InvoiceGenerator.php

namespace App\Invoicing;

use App\Billing\BillingStrategyResolver;
use App\Enums\BillingType;
use App\Enums\InvoiceStatus;
use App\Models\Client;
use App\Models\Invoice;
use App\Models\InvoiceLine;
use App\Models\Project;
use App\Models\TimeEntry;
use App\Support\DueDateCalculator;
use App\Support\Money;
use App\Support\TimeRounding;
use Carbon\CarbonImmutable;
use Illuminate\Database\Eloquent\Collection;
use Illuminate\Support\Facades\DB;

/**
 * Turns a client's billable, un-invoiced time entries into one draft invoice.
 *
 * Everything happens in one database transaction: if any rule fails, nothing is saved.
 */
final class InvoiceGenerator
{
    public function __construct(
        private readonly OverlapDetector $overlapDetector,
        private readonly BillingStrategyResolver $strategies,
        private readonly InvoiceNumberGenerator $numbers,
        private readonly DueDateCalculator $dueDates,
    ) {}

    /**
     * @throws NothingToInvoiceException when there are no entries to invoice
     * @throws OverlappingEntriesException when two of the entries overlap (R10)
     */
    public function generate(Client $client, CarbonImmutable $upTo): Invoice
    {
        return DB::transaction(function () use ($client, $upTo): Invoice {
            $entries = $this->invoiceableEntries($client, $upTo);

            if ($entries->isEmpty()) {
                throw NothingToInvoiceException::for($client, $upTo);
            }

            $this->assertNoOverlaps($entries);

            $lines = $this->buildLines($entries);
            $subtotal = Money::sum(
                ...array_map(fn (array $line): Money => Money::fromCents($line['amount_cents']), $lines),
            );
            $vatRateBp = (int) config('billable.vat_rate_bp');
            $vat = $subtotal->percentageBp($vatRateBp);
            $issuedOn = today()->toImmutable();

            $invoice = $client->invoices()->create([
                'number' => $this->numbers->next($issuedOn->year),
                'status' => InvoiceStatus::Draft,
                'issued_on' => $issuedOn,
                'due_on' => $this->dueDates->dueDate($issuedOn),
                'vat_rate_bp' => $vatRateBp,
                'subtotal_cents' => $subtotal->cents,
                'vat_cents' => $vat->cents,
                'total_cents' => $subtotal->add($vat)->cents,
            ]);

            $invoice->lines()->createMany($lines);

            TimeEntry::query()
                ->whereKey($entries->modelKeys())
                ->update(['invoice_id' => $invoice->id]);

            return $invoice->load('lines.project');
        });
    }

    /**
     * @return Collection<int, TimeEntry>
     */
    private function invoiceableEntries(Client $client, CarbonImmutable $upTo): Collection
    {
        return TimeEntry::query()
            ->with('project')
            ->whereIn('project_id', $client->projects()
                ->where('billing_type', '!=', BillingType::Internal)
                ->select('id'))
            ->whereNull('invoice_id')
            ->where('billable', true)
            ->where('ended_at', '<=', $upTo->endOfDay())
            ->orderBy('started_at')
            ->get();
    }

    /**
     * @param  Collection<int, TimeEntry>  $entries
     */
    private function assertNoOverlaps(Collection $entries): void
    {
        $slots = $entries->map(fn (TimeEntry $entry): TimeSlot => new TimeSlot(
            $entry->id,
            $entry->started_at->toImmutable(),
            $entry->ended_at->toImmutable(),
            $entry->description,
        ))->all();

        $overlap = $this->overlapDetector->find($slots);

        if ($overlap !== null) {
            throw new OverlappingEntriesException($overlap[0], $overlap[1]);
        }
    }

    /**
     * One line per project: rounded minutes and the amount from the project's billing strategy.
     *
     * @param  Collection<int, TimeEntry>  $entries
     * @return list<array{project_id: int, description: string, minutes: int, amount_cents: int}>
     */
    private function buildLines(Collection $entries): array
    {
        $alreadyBilled = $this->alreadyBilledCents($entries->pluck('project_id')->unique()->values()->all());
        $lines = [];

        foreach ($entries->groupBy('project_id') as $projectId => $projectEntries) {
            /** @var Project $project */
            $project = $projectEntries->first()->project;

            $minutes = (int) $projectEntries->sum(fn (TimeEntry $entry): int => TimeRounding::roundUp(
                $entry->durationMinutes(),
                $project->rounding_minutes,
            ));

            $amount = $this->strategies->forProject($project)->calculate(
                $project,
                $minutes,
                Money::fromCents($alreadyBilled[$projectId] ?? 0),
            );

            if ($minutes === 0 && $amount->isZero()) {
                continue;
            }

            $lines[] = [
                'project_id' => $project->id,
                'description' => self::describe($project, $minutes),
                'minutes' => $minutes,
                'amount_cents' => $amount->cents,
            ];
        }

        return $lines;
    }

    /**
     * What has already been billed per project on earlier invoices (R7, R8).
     *
     * @param  list<int>  $projectIds
     * @return array<int, int> project id => cents
     */
    private function alreadyBilledCents(array $projectIds): array
    {
        return InvoiceLine::query()
            ->whereIn('project_id', $projectIds)
            ->groupBy('project_id')
            ->selectRaw('project_id, SUM(amount_cents) AS billed_cents')
            ->pluck('billed_cents', 'project_id')
            ->map(fn (mixed $cents): int => (int) $cents)
            ->all();
    }

    public static function describe(Project $project, int $minutes): string
    {
        return sprintf('%s — %d h %02d min', $project->name, intdiv($minutes, 60), $minutes % 60);
    }
}
```

**No `Internal` billing type?** If you did not do the lesson 08 basic task, your `BillingType` has no
`Internal` case. Then delete the line `->where('billing_type', '!=', BillingType::Internal)` in
`invoiceableEntries()` and the `use App\Enums\BillingType;` import — every project is billed.

**What happens** when the controller calls `generate($client, 2026-09-30)`:

1. `DB::transaction()` opens a transaction and runs the closure. If the closure returns normally,
   Laravel commits. If **any** exception is thrown inside, Laravel rolls back and re-throws it. The
   closure's return value (the invoice) becomes `generate()`'s return value.
2. `invoiceableEntries()` runs **one** query for the entries (plus one for their projects, because of
   `with('project')` — eager loading from lesson 06). The conditions are exactly the rules: this
   client's projects (`whereIn` with a sub-query) except internal ones (the lesson 08 Null Object type
   is never billed, so its entries stay out of invoices and stay editable), not invoiced yet, billable,
   and ended no later than 23:59:59 on the chosen day.
3. No entries → `NothingToInvoiceException`. Nothing was written yet, but the rollback would undo it
   anyway.
4. `assertNoOverlaps()` maps each entry to a `TimeSlot` and asks the detector. `toImmutable()` gives
   the detector copies it cannot change. An overlap throws `OverlappingEntriesException` with both slots.
5. `buildLines()` first loads what was already billed per project in **one** grouped query
   (`SELECT project_id, SUM(amount_cents) ... GROUP BY project_id`). `pluck('billed_cents', 'project_id')`
   turns the result into `[projectId => cents]`. The `(int)` cast matters: PostgreSQL returns `SUM()`
   of integers as a `bigint`/numeric value that PDO may give you as a string.
6. `groupBy('project_id')` splits the entries into one collection per project. For each project:
   - every entry is rounded **separately** with `TimeRounding::roundUp()` (R5) and the results are summed;
   - the resolver picks the strategy (lesson 08) and the strategy prices the minutes, taking into
     account what was already billed (R6–R8);
   - a line with 0 minutes **and** 0 € is skipped. A line with minutes but 0 € (a fixed-price project
     that is already fully billed) is kept, so the client sees the work was done.
7. The subtotal is the sum of the line amounts. `array_map()` turns each line's cents into `Money`, and
   `...` spreads that array into `Money::sum()` from lesson 11 — this is the call that method was
   built for. VAT is calculated **once**, on the subtotal (R3), with `percentageBp()` and the rate from
   `config('billable.vat_rate_bp')` (lesson 10). The rate is copied onto the invoice (R2), so a later
   config change does not change old invoices.
8. `today()` respects `travelTo()` in tests, so "today" is controllable. `DueDateCalculator::dueDate()`
   applies R12.
9. `$this->numbers->next(2026)` runs **inside** the transaction. If a later step fails, the invoice row
   is rolled back and the number is free again — no gap.
10. `$client->invoices()->create([...])` inserts the invoice with `client_id` filled in by the relation.
    `createMany($lines)` inserts every line with `invoice_id` filled in.
11. `whereKey($entries->modelKeys())->update([...])` marks all entries in **one** `UPDATE ... WHERE id IN
    (...)`. `modelKeys()` returns the ids of an Eloquent collection.
12. `load('lines.project')` loads the lines and their projects for the page that shows the result.

> **Durations.** We reuse `TimeEntry::durationMinutes()` from lesson 10 instead of `diffInMinutes()`.
> In Carbon 3 (Laravel 11 and newer) `diffInMinutes()` returns a `float` and is negative when the
> arguments are reversed; older tutorials (Carbon 2) show it returning an absolute `int`.

**Check it works** — in Tinker, with the seed data from earlier lessons:

```bash
php artisan tinker
> $client = App\Models\Client::has('projects.timeEntries')->first();
> $invoice = app(App\Invoicing\InvoiceGenerator::class)->generate($client, now()->toImmutable());
> $invoice->number;
= "2026-0001"
> $invoice->lines->pluck('description');
> $invoice->timeEntries()->count();
```

Run the `generate` line a second time: you get `NothingToInvoiceException`, because the entries are
now invoiced. That is correct behaviour. To start over: `php artisan migrate:fresh --seed`.

**Commit:** `feat(invoices): generate invoices in one transaction`

---

## Step 8 — `InvoiceService`: send, mark paid, delete a draft

Status changes and deletion are business rules too, so they do not belong in the controller.

```php
<?php

// app/Services/InvoiceService.php

namespace App\Services;

use App\Enums\InvoiceStatus;
use App\Invoicing\InvalidStatusTransitionException;
use App\Invoicing\InvoiceNotEditableException;
use App\Models\Invoice;
use Illuminate\Support\Facades\DB;

final class InvoiceService
{
    public function send(Invoice $invoice): void
    {
        $this->moveTo($invoice, InvoiceStatus::Sent);
    }

    public function markPaid(Invoice $invoice): void
    {
        $this->moveTo($invoice, InvoiceStatus::Paid);
    }

    /**
     * Deletes a draft and releases its time entries so they can be invoiced again (R13).
     */
    public function delete(Invoice $invoice): void
    {
        if (! $invoice->status->canEdit()) {
            throw InvoiceNotEditableException::for($invoice);
        }

        DB::transaction(function () use ($invoice): void {
            $invoice->timeEntries()->update(['invoice_id' => null]);
            $invoice->delete();
        });
    }

    private function moveTo(Invoice $invoice, InvoiceStatus $to): void
    {
        if (! $invoice->status->canTransitionTo($to)) {
            throw InvalidStatusTransitionException::between($invoice->status, $to);
        }

        $invoice->update(['status' => $to]);
    }
}
```

**What happens:**

1. `moveTo()` asks the enum whether the move is allowed. If not, it throws; nothing changes.
2. `delete()` checks `canEdit()` first: a sent or paid invoice is refused (R13).
3. Inside one transaction, `$invoice->timeEntries()->update([...])` runs
   `UPDATE time_entries SET invoice_id = NULL WHERE invoice_id = ?` — the entries are released. Then the
   invoice is deleted, and PostgreSQL deletes its lines (`ON DELETE CASCADE`). Because of
   `restrictOnDelete()` on `time_entries.invoice_id`, the delete would fail if we forgot the release.

### The R14 guard in `TimeEntryService`

Open `app/Services/TimeEntryService.php` (lesson 07). Add this private method at the bottom of the
class, and call it as the **first line** of `update()` and of `delete()`. (If you do not have `update()`
and `delete()` because you skipped lesson 07 step 9, read the box *"If you do not have delete yet"*
below first — it adds a minimal delete action with the guard already in place.)

```php
// app/Services/TimeEntryService.php (add the import at the top)
use App\Invoicing\TimeEntryLockedException;

// ...at the start of update(TimeEntry $entry, array $data) and delete(TimeEntry $entry):
$this->assertNotInvoiced($entry);

// ...new private method at the bottom of the class:
private function assertNotInvoiced(TimeEntry $entry): void
{
    if ($entry->isInvoiced()) {
        throw TimeEntryLockedException::for($entry);
    }
}
```

In `TimeEntryController`, wrap the service calls in `update()` and `destroy()` so the user sees the
message instead of an error page:

```php
// app/Http/Controllers/TimeEntryController.php (inside update() and destroy(); import InvoicingException)
try {
    $this->timeEntries->delete($timeEntry);   // or ->update($timeEntry, $request->validated())
} catch (InvoicingException $e) {
    return back()->with('error', $e->getMessage());
}
```

Finally, on the time sheet view, hide the edit and delete buttons for invoiced entries:

```blade
{{-- resources/views/time-entries/index.blade.php (around the edit/delete buttons of one row) --}}
@if ($entry->isInvoiced())
    <small>Invoiced</small>
@else
    {{-- your existing edit link and delete form --}}
@endif
```

#### If you do not have delete yet

Lesson 06 registered only `index`, `create` and `store` for time entries, and lesson 07 step 9 was only
for students who had added edit and delete themselves. Lesson 14 tests that an invoiced entry cannot be
deleted through `DELETE /time-entries/{time_entry}`, so add a minimal delete action now. (Editing stays
optional; R14 for `update()` works the same way once you add it.)

**1. Route** — in `routes/web.php`, add `destroy` to the lesson 06 line:

```php
// routes/web.php (replace the lesson 06 time-entries line; the pattern line goes above it, see lesson 04 step 7)
Route::pattern('time_entry', '[0-9]+');
Route::resource('time-entries', TimeEntryController::class)->only(['index', 'create', 'store', 'destroy']);
```

**2. Service** — add `delete()` to `TimeEntryService`, below `log()`, together with the guard method from
above (`use App\Invoicing\TimeEntryLockedException;` at the top):

```php
// app/Services/TimeEntryService.php — add below log()

    /**
     * Deletes a time entry that is not on an invoice yet.
     *
     * @throws TimeEntryLockedException when the entry is on an invoice (R14)
     */
    public function delete(TimeEntry $entry): void
    {
        $this->assertNotInvoiced($entry);

        // tag_time_entry has ON DELETE CASCADE, so the pivot rows go with the entry.
        $entry->delete();
    }

    private function assertNotInvoiced(TimeEntry $entry): void
    {
        if ($entry->isInvoiced()) {
            throw TimeEntryLockedException::for($entry);
        }
    }
```

**3. Controller** — add `destroy()` to `TimeEntryController` (imports: `App\Invoicing\InvoicingException`
and `App\Models\TimeEntry`):

```php
// app/Http/Controllers/TimeEntryController.php — add below store()

    public function destroy(TimeEntry $timeEntry): RedirectResponse
    {
        try {
            $this->timeEntries->delete($timeEntry);
        } catch (InvoicingException $e) {
            return back()->with('error', $e->getMessage());
        }

        return redirect()
            ->route('time-entries.index')
            ->with('status', 'Time entry deleted.');
    }
```

The resource route's parameter is `{time_entry}`; Laravel matches it to the parameter `$timeEntry`
(snake_case ↔ camelCase), so route model binding works.

**4. View** — in `resources/views/time-entries/index.blade.php` add an empty `<th></th>` as the last
header cell, change the empty row to `<td colspan="8">`, and add this as the last cell of each row:

```blade
{{-- resources/views/time-entries/index.blade.php (last <td> of each row) --}}
<td>
    @if ($entry->isInvoiced())
        <small>Invoiced</small>
    @else
        <form method="POST" action="{{ route('time-entries.destroy', $entry) }}"
              data-confirm="Delete this time entry?"
              onsubmit="return confirm(this.dataset.confirm)">
            @csrf
            @method('DELETE')
            <button type="submit" class="secondary">Delete</button>
        </form>
    @endif
</td>
```

**Check it works:** `php artisan route:list --path=time-entries` shows a `DELETE` line
`time-entries.destroy`. Delete an entry that is not invoiced → "Time entry deleted." After you generate
an invoice (Step 10), its entries show *Invoiced* instead of the button.

**What happens:** the rule is enforced in the **service**, so it holds for every caller (a controller, a
command, a test). Hiding the buttons is only convenience; the guard is the real protection.

**Commit:** `feat(invoices): status transitions, draft deletion and R14 guard`

---

## Step 9 — Routes, form request and controller

```php
// routes/web.php (add below the other routes)
use App\Http\Controllers\InvoiceController;

Route::pattern('invoice', '[0-9]+');
Route::resource('invoices', InvoiceController::class)->only(['index', 'create', 'store', 'show', 'destroy']);
Route::post('invoices/{invoice}/send', [InvoiceController::class, 'send'])->name('invoices.send');
Route::post('invoices/{invoice}/mark-paid', [InvoiceController::class, 'markPaid'])->name('invoices.mark-paid');
```

```php
<?php

// app/Http/Requests/StoreInvoiceRequest.php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StoreInvoiceRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    /**
     * @return array<string, mixed>
     */
    public function rules(): array
    {
        return [
            'client_id' => ['required', 'integer', 'exists:clients,id'],
            'up_to' => ['required', 'date_format:Y-m-d', 'before_or_equal:today'],
        ];
    }
}
```

```php
<?php

// app/Http/Controllers/InvoiceController.php

namespace App\Http\Controllers;

use App\Http\Requests\StoreInvoiceRequest;
use App\Invoicing\InvoiceGenerator;
use App\Invoicing\InvoicingException;
use App\Models\Client;
use App\Models\Invoice;
use App\Services\InvoiceService;
use Carbon\CarbonImmutable;
use Illuminate\Http\RedirectResponse;
use Illuminate\View\View;

class InvoiceController extends Controller
{
    public function __construct(
        private readonly InvoiceGenerator $generator,
        private readonly InvoiceService $invoices,
    ) {}

    public function index(): View
    {
        $invoices = Invoice::query()
            ->with('client')
            ->latest('issued_on')
            ->latest('id')
            ->paginate(20);

        return view('invoices.index', ['invoices' => $invoices]);
    }

    public function create(): View
    {
        return view('invoices.create', [
            'clients' => Client::query()->orderBy('name')->get(['id', 'name']),
            'today' => today()->toDateString(),
        ]);
    }

    public function store(StoreInvoiceRequest $request): RedirectResponse
    {
        $client = Client::query()->findOrFail($request->integer('client_id'));

        try {
            $invoice = $this->generator->generate($client, CarbonImmutable::parse($request->validated('up_to')));
        } catch (InvoicingException $e) {
            return back()->withInput()->withErrors(['invoice' => $e->getMessage()]);
        }

        return redirect()
            ->route('invoices.show', $invoice)
            ->with('status', "Invoice {$invoice->number} was created as a draft.");
    }

    public function show(Invoice $invoice): View
    {
        $invoice->load(['client', 'lines.project']);

        return view('invoices.show', ['invoice' => $invoice]);
    }

    public function destroy(Invoice $invoice): RedirectResponse
    {
        try {
            $this->invoices->delete($invoice);
        } catch (InvoicingException $e) {
            return back()->with('error', $e->getMessage());
        }

        return redirect()
            ->route('invoices.index')
            ->with('status', "Draft {$invoice->number} was deleted. Its time entries can be invoiced again.");
    }

    public function send(Invoice $invoice): RedirectResponse
    {
        return $this->changeStatus(fn () => $this->invoices->send($invoice), $invoice, 'marked as sent');
    }

    public function markPaid(Invoice $invoice): RedirectResponse
    {
        return $this->changeStatus(fn () => $this->invoices->markPaid($invoice), $invoice, 'marked as paid');
    }

    private function changeStatus(callable $action, Invoice $invoice, string $done): RedirectResponse
    {
        try {
            $action();
        } catch (InvoicingException $e) {
            return back()->with('error', $e->getMessage());
        }

        return redirect()->route('invoices.show', $invoice)->with('status', "Invoice {$invoice->number} was {$done}.");
    }
}
```

**What happens:**

1. `resource(...)->only([...])` creates five routes: `GET /invoices`, `GET /invoices/create`,
   `POST /invoices`, `GET /invoices/{invoice}`, `DELETE /invoices/{invoice}`. There is no edit form
   today. `send` and `mark-paid` are `POST` because they **change** data; a `GET` link could be triggered
   by a browser prefetch or a search bot.
2. The form request validates the **shape** of the input: the client exists, the date is a real date and
   not in the future. The **business** rules (nothing to invoice, overlaps) are checked by the generator.
3. `store()` catches only `InvoicingException`. `withErrors(['invoice' => ...])` puts the message into
   the `$errors` bag under the key `invoice`, and `withInput()` refills the form. Any other exception
   (a database failure) is not caught, so Laravel shows the error page and logs it.
4. On success we redirect to the invoice page (Post/Redirect/Get, lesson 05), so a browser refresh does
   not generate a second invoice.
5. `show()` eager-loads the client, lines and their projects: the page runs a fixed number of queries,
   however many lines there are.
6. `index()` uses `paginate(20)` and `with('client')` — no N+1 on the list page.

---

## Step 10 — Views

```blade
{{-- resources/views/invoices/index.blade.php --}}
@extends('layouts.app')

@section('title', 'Invoices')

@section('content')
    <h1>Invoices</h1>

    <p><a href="{{ route('invoices.create') }}" role="button">Generate invoice</a></p>

    @if ($invoices->isEmpty())
        <p>No invoices yet.</p>
    @else
        <table>
            <thead>
                <tr><th>Number</th><th>Client</th><th>Issued</th><th>Due</th><th>Status</th><th>Total</th></tr>
            </thead>
            <tbody>
                @foreach ($invoices as $invoice)
                    <tr>
                        <td><a href="{{ route('invoices.show', $invoice) }}">{{ $invoice->number }}</a></td>
                        <td>{{ $invoice->client->name }}</td>
                        <td>{{ $invoice->issued_on->format('d.m.Y') }}</td>
                        <td>{{ $invoice->due_on->format('d.m.Y') }}</td>
                        <td>{{ $invoice->status->label() }}</td>
                        <td>{{ $invoice->total()->format() }}</td>
                    </tr>
                @endforeach
            </tbody>
        </table>

        {{ $invoices->links() }}
    @endif
@endsection
```

```blade
{{-- resources/views/invoices/create.blade.php --}}
@extends('layouts.app')

@section('title', 'Generate invoice')

@section('content')
    <h1>Generate invoice</h1>

    @error('invoice')
        <article role="alert"><strong>Not generated.</strong> {{ $message }}</article>
    @enderror

    <form method="POST" action="{{ route('invoices.store') }}">
        @csrf

        <label for="client_id">Client</label>
        <select id="client_id" name="client_id" required>
            <option value="">Choose a client…</option>
            @foreach ($clients as $client)
                <option value="{{ $client->id }}" @selected(old('client_id') == $client->id)>{{ $client->name }}</option>
            @endforeach
        </select>
        @error('client_id') <small>{{ $message }}</small> @enderror

        <label for="up_to">Include entries up to</label>
        <input id="up_to" name="up_to" type="date" value="{{ old('up_to', $today) }}" max="{{ $today }}" required>
        @error('up_to') <small>{{ $message }}</small> @enderror

        <button type="submit">Generate</button>
    </form>
@endsection
```

```blade
{{-- resources/views/invoices/show.blade.php --}}
@extends('layouts.app')

@section('title', 'Invoice '.$invoice->number)

@section('content')
    <h1>Invoice {{ $invoice->number }} <small>({{ $invoice->status->label() }})</small></h1>

    <p>
        <strong>{{ $invoice->client->name }}</strong><br>
        Issued {{ $invoice->issued_on->format('d.m.Y') }} · Due {{ $invoice->due_on->format('d.m.Y') }}
    </p>

    <table>
        <thead>
            <tr><th>Description</th><th>Minutes</th><th style="text-align: right">Amount</th></tr>
        </thead>
        <tbody>
            @foreach ($invoice->lines as $line)
                <tr>
                    <td>{{ $line->description }}</td>
                    <td>{{ $line->minutes }}</td>
                    <td style="text-align: right">{{ $line->amount()->format() }}</td>
                </tr>
            @endforeach
        </tbody>
        <tfoot>
            <tr><td colspan="2">Subtotal</td><td style="text-align: right">{{ $invoice->subtotal()->format() }}</td></tr>
            <tr><td colspan="2">VAT {{ $invoice->vat_rate_bp / 100 }} %</td><td style="text-align: right">{{ $invoice->vat()->format() }}</td></tr>
            <tr><th colspan="2">Total</th><th style="text-align: right">{{ $invoice->total()->format() }}</th></tr>
        </tfoot>
    </table>

    <div role="group">
        @if ($invoice->status->canTransitionTo(\App\Enums\InvoiceStatus::Sent))
            <form method="POST" action="{{ route('invoices.send', $invoice) }}">
                @csrf
                <button type="submit">Mark as sent</button>
            </form>
        @endif

        @if ($invoice->status->canTransitionTo(\App\Enums\InvoiceStatus::Paid))
            <form method="POST" action="{{ route('invoices.mark-paid', $invoice) }}">
                @csrf
                <button type="submit">Mark as paid</button>
            </form>
        @endif

        @if ($invoice->status->canEdit())
            <form method="POST" action="{{ route('invoices.destroy', $invoice) }}"
                  onsubmit="return confirm('Delete this draft and release its time entries?')">
                @csrf
                @method('DELETE')
                <button type="submit" class="secondary">Delete draft</button>
            </form>
        @endif
    </div>

    <p><a href="{{ route('invoices.index') }}">← All invoices</a></p>
@endsection
```

Add a link to `/invoices` in the navigation of `resources/views/layouts/app.blade.php`, next to the
other links: `<li><a href="{{ route('invoices.index') }}">Invoices</a></li>`.

**What happens:**

1. `{{ }}` escapes everything (lesson 04), so a project name with `<script>` is shown as text. Messages
   flashed with `->with('status', ...)` and `->with('error', ...)` are shown by the layout (lesson 05).
2. The buttons ask the enum (`canTransitionTo()`, `canEdit()`) — the view contains no status rules of its
   own. `vat_rate_bp / 100` shows `24` for display only; no money is calculated in the view.
3. Every form has `@csrf`. The delete form uses `@method('DELETE')` because HTML forms only know GET and
   POST.

**Check it works:**

1. `php artisan migrate:fresh --seed`, then open `/invoices/create`, pick a client with entries, keep
   today's date, press *Generate*. You land on `/invoices/1` with number `2026-0001` and one line per
   project.
2. Press *Mark as sent*. The *Delete draft* button disappears. Try to edit one of its entries on the
   time sheet: the buttons are gone (R14).
3. Create an overlap in Tinker and try again:

```bash
php artisan tinker
> $p = App\Models\Project::first();
> App\Models\TimeEntry::create(['project_id' => $p->id, 'started_at' => '2026-09-29 13:00', 'ended_at' => '2026-09-29 17:00', 'description' => 'API integration', 'billable' => true]);
> App\Models\TimeEntry::create(['project_id' => $p->id, 'started_at' => '2026-09-29 13:30', 'ended_at' => '2026-09-29 14:00', 'description' => 'Client call', 'billable' => true]);
```

Generate an invoice for that project's client: the form shows *"Cannot generate the invoice: entry #…
"API integration" (29.09.2026 13:00–17:00) overlaps entry #… "Client call" (29.09.2026 13:30–14:00)…"*.
(`TimeEntry::create()` works because these fields are in `$fillable` since lesson 06.)

**Commit:** `feat(invoices): invoice pages and actions`

---

## Step 11 — A feature test for the generator

Unit tests proved the pieces. Now one test proves they work **together**, with a real database, real
models and a controlled date. We use `Tests\TestCase` (Laravel booted), `RefreshDatabase` (every test
starts with an empty, migrated database), factories and `travelTo()`.

### First: factories for time entries and invoices

Lesson 03 gave you `ClientFactory` and `ProjectFactory`. Tests from now on also need time entries and
invoices, so we add a factory for each instead of writing the same attribute arrays in every test.

```bash
php artisan make:factory TimeEntryFactory --model=TimeEntry
php artisan make:factory InvoiceFactory --model=Invoice
```

`Invoice` already uses `HasFactory` (Step 3). `TimeEntry` from lesson 06 does not yet — add the trait:

```php
// app/Models/TimeEntry.php (add the imports at the top and the trait inside the class)
use Database\Factories\TimeEntryFactory;
use Illuminate\Database\Eloquent\Factories\HasFactory;

class TimeEntry extends Model
{
    /** @use HasFactory<TimeEntryFactory> */
    use HasFactory;

    // ... the rest stays as it is
}
```

```php
<?php

// database/factories/TimeEntryFactory.php

namespace Database\Factories;

use App\Models\Project;
use App\Models\TimeEntry;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<TimeEntry>
 */
class TimeEntryFactory extends Factory
{
    /**
     * By default: one billable hour on a day in the last 30 days, on a new project.
     *
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        $start = now()->subDays(fake()->numberBetween(1, 30))->setTime(fake()->numberBetween(8, 16), 0);

        return [
            'project_id' => Project::factory(),
            'started_at' => $start,
            'ended_at' => $start->copy()->addHour(),
            'description' => fake()->sentence(3),
            'billable' => true,
        ];
    }

    public function between(string $start, string $end): static
    {
        return $this->state(fn () => [
            'started_at' => $start,
            'ended_at' => $end,
        ]);
    }

    public function notBillable(): static
    {
        return $this->state(fn () => [
            'billable' => false,
        ]);
    }
}
```

```php
<?php

// database/factories/InvoiceFactory.php

namespace Database\Factories;

use App\Enums\InvoiceStatus;
use App\Models\Client;
use App\Models\Invoice;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<Invoice>
 */
class InvoiceFactory extends Factory
{
    /**
     * By default: a draft of 100.00 € + 24 % VAT for a new client.
     *
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        return [
            'client_id' => Client::factory(),
            'number' => sprintf('2026-%04d', fake()->unique()->numberBetween(1, 9999)),
            'status' => InvoiceStatus::Draft,
            'issued_on' => '2026-09-30',
            'due_on' => '2026-10-14',
            'vat_rate_bp' => 2400,
            'subtotal_cents' => 10000,
            'vat_cents' => 2400,
            'total_cents' => 12400,
        ];
    }

    public function sent(): static
    {
        return $this->state(fn () => [
            'status' => InvoiceStatus::Sent,
        ]);
    }

    public function paid(): static
    {
        return $this->state(fn () => [
            'status' => InvoiceStatus::Paid,
        ]);
    }
}
```

**What happens:**

1. `HasFactory` with the `@use` comment (the same pattern as lesson 03) makes `TimeEntry::factory()`
   work and tells your editor which factory it returns.
2. `TimeEntryFactory` uses `now()`, not a fixed date, so a default entry is always in the past —
   also after `travelTo()`. When the exact time matters (rounding, overlaps), say it with the
   `between()` state. Two default entries on the same project **can** overlap by chance, so tests of
   the generator always use `between()`.
3. `'project_id' => Project::factory()` creates a new project (and client) for every entry, unless you
   pass one with `->for($project)`.
4. `InvoiceFactory` builds a **finished-looking** invoice row directly. It does not go through
   `InvoiceGenerator`, so it has no lines and no linked entries, and its number does not come from
   `InvoiceNumberGenerator`. Use it when a test needs "an invoice in status X" (Task 1, lesson 14);
   use the generator when the test is **about** generating.
5. `fake()->unique()` keeps the numbers different inside one test, so `count(30)` does not hit the
   unique index. The states `sent()` and `paid()` change only the status, like `archived()` in lesson 03.
6. Factories are for tests and seeders. Production code never calls them.

### The test

```php
<?php

// tests/Feature/Invoicing/InvoiceGeneratorTest.php

namespace Tests\Feature\Invoicing;

use App\Enums\BillingType;
use App\Enums\InvoiceStatus;
use App\Invoicing\InvoiceGenerator;
use App\Invoicing\NothingToInvoiceException;
use App\Invoicing\OverlappingEntriesException;
use App\Models\Client;
use App\Models\Project;
use App\Models\TimeEntry;
use Carbon\CarbonImmutable;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class InvoiceGeneratorTest extends TestCase
{
    use RefreshDatabase;

    private Client $client;

    private Project $project;

    protected function setUp(): void
    {
        parent::setUp();

        $this->travelTo(CarbonImmutable::parse('2026-09-30 12:00'));

        $this->client = Client::factory()->create();
        $this->project = Project::factory()->for($this->client)->create([
            'name' => 'Website redesign',
            'billing_type' => BillingType::Hourly,
            'hourly_rate_cents' => 6000,
            'fixed_price_cents' => null,
            'budget_cap_cents' => null,
            'rounding_minutes' => 15,
        ]);
    }

    public function test_it_creates_a_draft_with_rounded_lines_vat_and_linked_entries(): void
    {
        $first = $this->entry('2026-09-28 09:00', '2026-09-28 10:10');   // 70 min → 75
        $second = $this->entry('2026-09-29 13:00', '2026-09-29 14:00');  // 60 min → 60
        $notBillable = TimeEntry::factory()->for($this->project)->notBillable()
            ->between('2026-09-29 15:00', '2026-09-29 16:00')->create();
        $tooLate = $this->entry('2026-10-01 09:00', '2026-10-01 10:00');

        $invoice = $this->generator()->generate($this->client, CarbonImmutable::parse('2026-09-30'));

        $this->assertSame('2026-0001', $invoice->number);
        $this->assertSame(InvoiceStatus::Draft, $invoice->status);
        $this->assertSame('2026-09-30', $invoice->issued_on->toDateString());
        $this->assertSame('2026-10-14', $invoice->due_on->toDateString());

        $this->assertCount(1, $invoice->lines);
        $line = $invoice->lines->first();
        $this->assertSame('Website redesign — 2 h 15 min', $line->description);
        $this->assertSame(135, $line->minutes);
        $this->assertSame(13500, $line->amount_cents);          // 135 min × 60.00 €/h

        $this->assertSame(13500, $invoice->subtotal_cents);
        $this->assertSame(3240, $invoice->vat_cents);           // 24 %, rounded half up
        $this->assertSame(16740, $invoice->total_cents);

        $this->assertSame($invoice->id, $first->fresh()->invoice_id);
        $this->assertSame($invoice->id, $second->fresh()->invoice_id);
        $this->assertNull($notBillable->fresh()->invoice_id);
        $this->assertNull($tooLate->fresh()->invoice_id);
    }

    public function test_the_second_invoice_of_the_year_gets_the_next_number(): void
    {
        $this->entry('2026-09-28 09:00', '2026-09-28 10:00');
        $this->generator()->generate($this->client, CarbonImmutable::parse('2026-09-28'));

        $this->entry('2026-09-29 09:00', '2026-09-29 10:00');
        $second = $this->generator()->generate($this->client, CarbonImmutable::parse('2026-09-30'));

        $this->assertSame('2026-0002', $second->number);
    }

    public function test_overlapping_entries_are_rejected_and_named(): void
    {
        $this->entry('2026-09-29 13:00', '2026-09-29 17:00', ['description' => 'API integration']);
        $this->entry('2026-09-29 13:30', '2026-09-29 14:00', ['description' => 'Client call']);

        try {
            $this->generator()->generate($this->client, CarbonImmutable::parse('2026-09-30'));
            $this->fail('Expected OverlappingEntriesException.');
        } catch (OverlappingEntriesException $e) {
            $this->assertStringContainsString('API integration', $e->getMessage());
            $this->assertStringContainsString('Client call', $e->getMessage());
            $this->assertStringContainsString('29.09.2026 13:30', $e->getMessage());
        }

        $this->assertDatabaseCount('invoices', 0);
        $this->assertSame(0, TimeEntry::query()->whereNotNull('invoice_id')->count());
    }

    public function test_nothing_to_invoice_is_reported(): void
    {
        $this->expectException(NothingToInvoiceException::class);

        $this->generator()->generate($this->client, CarbonImmutable::parse('2026-09-30'));
    }

    private function generator(): InvoiceGenerator
    {
        return $this->app->make(InvoiceGenerator::class);
    }

    /**
     * @param  array<string, mixed>  $overrides
     */
    private function entry(string $start, string $end, array $overrides = []): TimeEntry
    {
        return TimeEntry::factory()->for($this->project)->between($start, $end)->create($overrides);
    }
}
```

**What happens:**

1. `travelTo()` freezes "now" at 30 September 2026, 12:00. `today()` inside the generator returns that
   date, so the issue date, the year of the number and the due date are predictable.
2. `RefreshDatabase` migrates once and wraps each test in a transaction that is rolled back at the end.
   Each test starts empty — so the first number is always `2026-0001`.
3. The first test checks every rule we implemented, with numbers you can recalculate by hand:
   70 → 75 minutes (15-minute rounding), 75 + 60 = 135 minutes, 135 × 6000 / 60 = 13,500 cents,
   VAT (13,500 × 2400 + 5000) / 10000 = 3,240 cents.
4. `fresh()` reloads an entry from the database, so we see what the `UPDATE` really wrote.
5. The overlap test checks the message names both entries **and** that nothing was saved.
6. The `entry()` helper is one factory call: `for($this->project)` fills in `project_id`,
   `between()` sets the times, and `create($overrides)` overrides anything else (here: the
   description in the overlap test). The not-billable entry uses the `notBillable()` state instead.

**Check it works:**

```bash
php artisan test
```

All tests pass, including the ones from lessons 11 and 12. Then run Pint: `./vendor/bin/pint --test`.

> **SQLite or PostgreSQL?** The default Laravel `phpunit.xml` runs tests on SQLite in memory. This test
> passes on both. Lesson 14 explains the trade-off and shows how to run tests on PostgreSQL.

**Commit:** `test(invoices): feature test for invoice generation`

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `Call to undefined method App\Models\Client::invoices()` | Relation missing | Add `invoices()` to `Client` (Step 3) |
| `Add [client_id] to fillable property to allow mass assignment on [App\Models\Invoice]` | `$fillable` incomplete | Copy the `$fillable` list from Step 3 |
| `column "updated_at" of relation "invoice_lines" does not exist` | Eloquent writes timestamps | `public $timestamps = false;` in `InvoiceLine` |
| `Attempted to lazy load [project] on model [App\Models\TimeEntry] but lazy loading is disabled` | `with('project')` missing in the entries query | Keep `->with('project')` in `invoiceableEntries()` — this is lesson 06's guard doing its job |
| `SQLSTATE[23503]: Foreign key violation … time_entries_invoice_id_foreign` when deleting an invoice | Entries still point to it | Delete through `InvoiceService::delete()`, which releases them first |
| `SQLSTATE[23505]: Unique violation … invoices_number_unique` | Two invoices generated at the same moment (two tabs, double click) | Expected with max + 1 — lesson 14 fixes it properly |
| Durations are 0 or negative | `diffInMinutes()` used instead of `durationMinutes()` (Carbon 3 returns a signed float) | Use `$entry->durationMinutes()` from lesson 10 |
| `A facade root has not been set.` in a unit test | A class under test uses `DB::`/`Invoice::` but the test extends `PHPUnit\Framework\TestCase` | Only pure classes get pure unit tests; test the rest in `tests/Feature` |
| `Call to undefined method App\Models\TimeEntry::factory()` | `use HasFactory;` missing in `TimeEntry` | Add the trait (Step 11) |
| `Unable to locate factory for [App\Models\Invoice]` | `database/factories/InvoiceFactory.php` missing or in the wrong namespace | Create it with `php artisan make:factory InvoiceFactory --model=Invoice` (Step 11) |
| A generator test fails now and then with `OverlappingEntriesException` | Two entries with the factory's random default times overlapped | Give every entry fixed times with `between()` |
| `Call to undefined method App\Billing\BillingStrategyResolver::for()` | Old name from a tutorial or AI tool | The method is `forProject()` (lesson 08) |
| `UnhandledMatchError` | A new enum case without a `match` arm | Handle the case in every `match` of `InvoiceStatus` |

---

## Recap

- **Migrations:** `invoices`, `invoice_lines` (cascade delete), `time_entries.invoice_id` (restrict delete).
- **Models:** `Invoice`, `InvoiceLine`, `InvoiceStatus` enum; relations on `Client` and `TimeEntry`.
- **Pure pieces:** `TimeSlot`, `OverlapDetector` (sort + sweep, O(n log n)), `InvoiceNumberGenerator::format()` / `sequenceOf()` — all with data-provider unit tests.
- **Orchestrator:** `InvoiceGenerator::generate()` — load, check overlaps, round per entry, price per project, VAT once, number, due date, save, mark entries — inside `DB::transaction()`.
- **Rules as exceptions:** `InvoicingException` and its children carry messages the user can act on.
- **Services:** `InvoiceService` (send, mark paid, delete draft and release entries) and the R14 guard in `TimeEntryService`.
- **HTTP:** `InvoiceController`, `StoreInvoiceRequest`, three Blade views, POST for every change.
- **Factories:** `TimeEntryFactory` (`between()`, `notBillable()`) and `InvoiceFactory` (`sent()`, `paid()`).
- **Feature test** with `RefreshDatabase`, factories and `travelTo()`.

---

## Independent work — solutions

<details>
<summary>Task 1 — Complete the status transitions (Basic)</summary>

Move the rule into the model, so every caller uses the same method:

```php
// app/Models/Invoice.php (add inside the class; import InvalidStatusTransitionException)
public function transitionTo(InvoiceStatus $to): void
{
    if (! $this->status->canTransitionTo($to)) {
        throw InvalidStatusTransitionException::between($this->status, $to);
    }

    $this->update(['status' => $to]);
}
```

`InvoiceService::moveTo()` becomes one line: `$invoice->transitionTo($to);`.

The enum is pure, so its test is a pure unit test with all 9 combinations:

```php
<?php

// tests/Unit/Invoicing/InvoiceStatusTest.php

namespace Tests\Unit\Invoicing;

use App\Enums\InvoiceStatus;
use PHPUnit\Framework\Attributes\DataProvider;
use PHPUnit\Framework\TestCase;

final class InvoiceStatusTest extends TestCase
{
    /** @return array<string, array{InvoiceStatus, InvoiceStatus, bool}> */
    public static function transitions(): array
    {
        return [
            'draft → draft' => [InvoiceStatus::Draft, InvoiceStatus::Draft, false],
            'draft → sent' => [InvoiceStatus::Draft, InvoiceStatus::Sent, true],
            'draft → paid' => [InvoiceStatus::Draft, InvoiceStatus::Paid, false],
            'sent → draft' => [InvoiceStatus::Sent, InvoiceStatus::Draft, false],
            'sent → sent' => [InvoiceStatus::Sent, InvoiceStatus::Sent, false],
            'sent → paid' => [InvoiceStatus::Sent, InvoiceStatus::Paid, true],
            'paid → draft' => [InvoiceStatus::Paid, InvoiceStatus::Draft, false],
            'paid → sent' => [InvoiceStatus::Paid, InvoiceStatus::Sent, false],
            'paid → paid' => [InvoiceStatus::Paid, InvoiceStatus::Paid, false],
        ];
    }

    #[DataProvider('transitions')]
    public function test_only_draft_to_sent_and_sent_to_paid_are_allowed(InvoiceStatus $from, InvoiceStatus $to, bool $allowed): void
    {
        $this->assertSame($allowed, $from->canTransitionTo($to));
    }

    /** @return array<string, array{InvoiceStatus, bool}> */
    public static function editable(): array
    {
        return [
            'draft' => [InvoiceStatus::Draft, true],
            'sent' => [InvoiceStatus::Sent, false],
            'paid' => [InvoiceStatus::Paid, false],
        ];
    }

    #[DataProvider('editable')]
    public function test_only_drafts_can_be_edited(InvoiceStatus $status, bool $expected): void
    {
        $this->assertSame($expected, $status->canEdit());
    }
}
```

And one feature test that a forbidden move changes nothing:

```php
<?php

// tests/Feature/Invoicing/InvoiceTransitionTest.php

namespace Tests\Feature\Invoicing;

use App\Enums\InvoiceStatus;
use App\Invoicing\InvalidStatusTransitionException;
use App\Models\Invoice;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class InvoiceTransitionTest extends TestCase
{
    use RefreshDatabase;

    public function test_a_draft_cannot_be_marked_paid(): void
    {
        $invoice = Invoice::factory()->create(); // a draft

        $this->expectException(InvalidStatusTransitionException::class);

        try {
            $invoice->transitionTo(InvoiceStatus::Paid);
        } finally {
            $this->assertSame(InvoiceStatus::Draft, Invoice::find($invoice->id)->status);
        }
    }
}
```

The show page from Step 10 already offers only allowed buttons, because it asks `canTransitionTo()`.
</details>

<details>
<summary>Task 2 — <code>docs/algorithm.md</code> (Basic)</summary>

A good example. Write yours in your own words and point to your own test names.

```markdown
# Algorithm: overlap detection in invoice generation

## Problem
Before an invoice is generated, the included time entries must not overlap in time (rule R10), because
one person cannot work on two things at once. Input: a list of entries (start, end, description) in any
order. Output: none, or one pair of entries that overlap, so the error can name both.

## Approach
Sort the entries by start time, and by end time when starts are equal. Walk through them once and
remember the entry with the latest end seen so far. If the next entry starts before that end, the two
overlap. Otherwise, if the next entry ends later, it becomes the new "latest".

## Why it is correct
If X and Y overlap and X starts first, then when Y is reached the latest end seen so far is at least
X's end, which is after Y's start — so the check fires. The pair reported is always a real overlap,
because "latest" started no later than Y and ends after Y starts.

## Complexity
Time O(n log n): the sort dominates; the sweep is O(n). Extra space O(n) for the sorted copy.

## Rejected alternative
Comparing every pair (two nested loops) is simpler to read, but O(n²): 49,995,000 comparisons for
10,000 entries instead of about 143,000. We also rejected "compare only with the previous entry":
it answers yes/no correctly but misses conflicts behind a long entry as soon as we report more than
the first overlap.

## Edge cases (tests in tests/Unit/Invoicing/OverlapDetectorTest.php)
- no entries, one entry → no overlap (`no slots`, `one slot`)
- touching entries (10:00–10:00) are not an overlap (`touching slots are allowed`)
- same start time (`same start time`)
- one entry inside another (`nested slot`)
- a chain of touching entries followed by an overlap (`chain of touching slots, then an overlap`)
- input not sorted (`unsorted input`)
- same hours on different days (`same hours on different days`)

## Other decisions
An invoice whose total is 0 € (e.g. only a fully billed fixed-price project) is allowed and shows the
work that was done.
```
</details>

<details>
<summary>Task 3 — Discount with largest-remainder allocation (Intermediate)</summary>

**1. `Money::allocate()`.** Add this method to `app/Support/Money.php`:

```php
// app/Support/Money.php (add inside the class)
/**
 * Splits this amount into parts proportional to the ratios, so that the parts add up to exactly this
 * amount (largest remainder method).
 *
 * @param  list<int>  $ratios  non-negative, at least one greater than zero
 * @return list<Money>
 */
public function allocate(array $ratios): array
{
    if ($ratios === [] || min($ratios) < 0 || array_sum($ratios) === 0) {
        throw new \InvalidArgumentException('Ratios must be non-negative and not all zero.');
    }
    if ($this->cents < 0) {
        throw new \InvalidArgumentException('Only non-negative amounts can be allocated.');
    }

    $total = array_sum($ratios);
    $shares = [];
    $remainders = [];

    foreach ($ratios as $i => $ratio) {
        $exact = $this->cents * $ratio;          // integer; the "true" share is $exact / $total
        $shares[$i] = intdiv($exact, $total);    // rounded down
        $remainders[$i] = $exact % $total;       // what rounding down threw away
    }

    $left = $this->cents - array_sum($shares);   // always smaller than count($ratios)
    $order = array_keys($ratios);
    usort($order, fn (int $a, int $b): int => [$remainders[$b], $a] <=> [$remainders[$a], $b]);

    for ($k = 0; $k < $left; $k++) {
        $shares[$order[$k]]++;
    }

    return array_map(fn (int $cents): self => self::fromCents($cents), array_values($shares));
}
```

How it works: every part first gets its share **rounded down**. The cents lost by rounding down are
`$left`, and there are always fewer of them than parts. They go, one each, to the parts with the largest
remainders (ties: the earlier part). Everything stays in integers — no float anywhere.

```php
<?php

// tests/Unit/Support/MoneyAllocateTest.php

namespace Tests\Unit\Support;

use App\Support\Money;
use InvalidArgumentException;
use PHPUnit\Framework\Attributes\DataProvider;
use PHPUnit\Framework\TestCase;

final class MoneyAllocateTest extends TestCase
{
    /** @return array<string, array{int, list<int>, list<int>}> */
    public static function cases(): array
    {
        return [
            'equal split with a leftover cent' => [100, [1, 1, 1], [34, 33, 33]],
            'exact split' => [300, [1, 1, 1], [100, 100, 100]],
            'uneven ratios' => [603, [1005, 2010, 3015], [101, 201, 301]],
            'a zero ratio gets nothing' => [100, [1, 0, 1], [50, 0, 50]],
            'fewer cents than parts' => [2, [1, 1, 1], [1, 1, 0]],
            'nothing to split' => [0, [3, 7], [0, 0]],
        ];
    }

    /**
     * @param  list<int>  $ratios
     * @param  list<int>  $expected
     */
    #[DataProvider('cases')]
    public function test_it_allocates_by_largest_remainder(int $cents, array $ratios, array $expected): void
    {
        $parts = Money::fromCents($cents)->allocate($ratios);

        $this->assertSame($expected, array_map(fn (Money $m): int => $m->cents, $parts));
        $this->assertSame($cents, Money::sum(...$parts)->cents); // the parts add up to the whole
    }

    public function test_all_zero_ratios_are_refused(): void
    {
        $this->expectException(InvalidArgumentException::class);

        Money::fromCents(100)->allocate([0, 0]);
    }
}
```

**2. Migration.** `php artisan make:migration add_discount_to_invoices`:

```php
<?php

// database/migrations/xxxx_xx_xx_xxxxxx_add_discount_to_invoices.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('invoices', function (Blueprint $table) {
            $table->integer('discount_bp')->default(0);
        });
        Schema::table('invoice_lines', function (Blueprint $table) {
            $table->bigInteger('discount_cents')->default(0);
        });
    }

    public function down(): void
    {
        Schema::table('invoice_lines', fn (Blueprint $table) => $table->dropColumn('discount_cents'));
        Schema::table('invoices', fn (Blueprint $table) => $table->dropColumn('discount_bp'));
    }
};
```

Add `'discount_bp'` to `Invoice::$fillable` and `'discount_cents'` to `InvoiceLine::$fillable`.

**3. Generator.** Change the signature to
`generate(Client $client, CarbonImmutable $upTo, int $discountBp = 0): Invoice` (add `$discountBp` to
the closure's `use`) and replace the subtotal part with:

```php
// app/Invoicing/InvoiceGenerator.php (inside the transaction, after $lines = $this->buildLines($entries);)
$gross = Money::sum(
    ...array_map(fn (array $line): Money => Money::fromCents($line['amount_cents']), $lines),
);
$discount = $gross->percentageBp($discountBp);

if (! $discount->isZero()) {
    $shares = $discount->allocate(array_column($lines, 'amount_cents'));
    foreach ($lines as $i => $line) {
        $lines[$i]['discount_cents'] = $shares[$i]->cents;
    }
}

$subtotal = $gross->subtract($discount);
$vatRateBp = (int) config('billable.vat_rate_bp');
$vat = $subtotal->percentageBp($vatRateBp);
```

…and add `'discount_bp' => $discountBp,` to the `create([...])` array. `amount_cents` keeps the
**undiscounted** line amount, so "already billed" for fixed and capped projects still measures the value
delivered; the discount is a reduction of the price, not of the work. Write that decision in
`docs/algorithm.md`.

**4. Form and controller.** In `StoreInvoiceRequest` add
`'discount_percent' => ['nullable', 'integer', 'between:0,100']`; in the create view add
`<input name="discount_percent" type="number" min="0" max="100" value="{{ old('discount_percent', 0) }}">`;
in `store()` pass `$request->integer('discount_percent') * 100` as the third argument. On the show page,
show a "Discount" row above the subtotal and each line's `discount_cents`.

**5. Test.** Extend the feature test: three projects whose lines are 10.05 €, 20.10 € and 30.15 €, discount 10 %.
Assert the line discounts are `[101, 201, 301]` and their sum `603` equals `percentageBp(1000)` of the
gross subtotal `6030`.
</details>

<details>
<summary>Task 4 — Measure naive vs sweep (Advanced)</summary>

The benchmark is a developer tool, so it lives in an Artisan command, not in the production classes.

```php
<?php

// app/Console/Commands/BenchmarkOverlaps.php

namespace App\Console\Commands;

use App\Invoicing\OverlapDetector;
use App\Invoicing\TimeSlot;
use DateTimeImmutable;
use Illuminate\Console\Command;
use Illuminate\Support\Benchmark;

class BenchmarkOverlaps extends Command
{
    protected $signature = 'invoices:benchmark-overlaps';

    protected $description = 'Compare naive O(n²) and sort-and-sweep overlap detection';

    public function handle(OverlapDetector $detector): int
    {
        $rows = [];

        foreach ([100, 1_000, 10_000] as $n) {
            $slots = $this->slots($n);

            $naiveMs = Benchmark::measure(fn () => $this->naive($slots));
            $sweepMs = Benchmark::measure(fn () => $detector->find($slots));

            $rows[] = [$n, number_format($n * ($n - 1) / 2), round($naiveMs, 2), round($sweepMs, 2)];
        }

        $this->table(['n', 'naive pairs', 'naive ms', 'sweep ms'], $rows);

        return self::SUCCESS;
    }

    /** @return list<TimeSlot> 45-minute slots with 15-minute gaps, shuffled, no overlaps */
    private function slots(int $n): array
    {
        $slots = [];
        $time = new DateTimeImmutable('2026-01-01 08:00');

        for ($i = 1; $i <= $n; $i++) {
            $end = $time->modify('+45 minutes');
            $slots[] = new TimeSlot($i, $time, $end, "Entry {$i}");
            $time = $end->modify('+15 minutes');
        }

        shuffle($slots);

        return $slots;
    }

    /** @param list<TimeSlot> $slots */
    private function naive(array $slots): ?array
    {
        $count = count($slots);

        for ($i = 0; $i < $count; $i++) {
            for ($j = $i + 1; $j < $count; $j++) {
                if ($slots[$i]->overlaps($slots[$j])) {
                    return [$slots[$i], $slots[$j]];
                }
            }
        }

        return null;
    }
}
```

`Benchmark::measure()` is Laravel's built-in stopwatch: it runs the closure and returns the time in
milliseconds (internally `hrtime()`, a monotonic nanosecond clock). Pass a second argument, e.g.
`Benchmark::measure(fn () => ..., iterations: 10)`, to get the average of several runs.

Run `php artisan invoices:benchmark-overlaps`. On the teacher's laptop (PHP 8.4):

| n | naive pairs | naive | sweep |
|---|---|---|---|
| 100 | 4,950 | 0.4 ms | 0.3 ms |
| 1,000 | 499,500 | 38 ms | 1.3 ms |
| 10,000 | 49,995,000 | 3,900 ms | 17 ms |

From 1,000 to 10,000 the naive time grows ×100 (n² grows ×100) and the sweep time about ×13 (n log n
grows about ×13). The data has **no overlaps on purpose**: with an overlap, both versions `return`
early, and the timing would depend on where the overlap happens to be — you would measure luck, not
the algorithm. Your numbers will differ; the growth pattern should not.
</details>
