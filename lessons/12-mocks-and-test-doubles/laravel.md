# 12 — Mocks and test doubles · Laravel

[← Concepts](README.md) · Starting point: end of lesson 11 · Estimated time in class: 2 h · Independent work tasks: [README](README.md#independent-work-2-h)

---

## What we build today

- `tests/Feature/Services/BudgetMonitorTest.php` — the budget warning (R9) with the `Notifier` replaced
  by a **Mockery mock**: sent at 80 %, not at 79 %, not when already sent, not for hourly projects, not
  twice, not when another request claimed it first, and the claim released when sending fails. Time is
  frozen with `$this->travelTo()`.
- One version of the same check with a **spy**, to see the difference.
- `tests/Fakes/InMemoryNotifier.php` — a hand-written **fake**, and a test that uses it.
- `tests/Feature/Services/TimeEntryServiceTest.php` — `Event::fake()` proves that logging an entry
  dispatches `TimeEntryLogged`, and that a rejected entry does not.
- `app/Mail/NotificationMail.php` — `MailNotifier` switches from `Mail::raw()` to a mailable, so that
  `Mail::fake()` can see it — and `tests/Feature/Notifiers/MailNotifierTest.php`.

**Final result:** `php artisan test` is green, including about 12 new feature tests. Each budget test
states one scenario of R9, and half of them prove that something did **not** happen.

### The code we test

These are the classes from lessons 07–11 that today's tests use. If your names differ slightly, adapt
the tests.

| Class | What the tests rely on |
|---|---|
| `App\Contracts\Notifier` | `notify(string $subject, string $message): void`; bound in `AppServiceProvider::register()` |
| `App\Notifiers\MailNotifier` | `__construct(#[Config('billable.owner_email')] string $ownerEmail)`; lesson 09 sends with `Mail::raw()` |
| `App\Services\BudgetMonitor` | `__construct(Notifier, HourlyBilling, ProjectService)`, `check(Project $project): void`; minutes from `ProjectService::billableMinutes()` (lesson 10); subject `"Budget warning: {name}"`; claims `budget_warning_sent_at = now()` with a conditional `UPDATE` before notifying, and sets it back to `null` if `notify()` throws |
| `App\Services\TimeEntryService` | `log(array $data): TimeEntry` with keys `project_id`, `started_at`, `ended_at`, `description`, `billable`, `tags`; throws `ProjectArchivedException` and `TimeEntryInFutureException` (lesson 07 intermediate task); dispatches `TimeEntryLogged` after the transaction |
| `App\Events\TimeEntryLogged` | `public TimeEntry $timeEntry` |
| Factories | `Client::factory()`, `Project::factory()` with its states (lesson 03). There is no `TimeEntryFactory` (lesson 06 did not create one), so tests create entries through the relation: `$project->timeEntries()->create([...])` |

---

## Step 1 — Feature tests and the test database

**Why:** `BudgetMonitor` reads a project and its time entries from the database and claims the warning
with an `UPDATE`. That is not pure code, so a plain PHPUnit unit test (lesson 11) cannot run it. We have two options:

| Option | How | Trade-off |
|---|---|---|
| **(a) Feature test** | Real Laravel app, real (test) database, **only the notifier replaced** | Realistic: the query, the rounding and the save are tested too. Slower (tens of ms per test) |
| **(b) Pure unit test** | Change `BudgetMonitor` so it receives plain values (minutes, rate, cap) instead of a model | Very fast, but the class changes shape for the test, and the query is no longer tested |

Option (b) would also need a second seam: the claim is an `UPDATE` query, so a pure unit test would have
to hide it behind another interface (for example a `BudgetWarningClaim` with a database
implementation). In Laravel, Eloquent models *are* the database access, so option (a) is the realistic
route. We choose it — and the claim query is tested for free. The important part stays the same: the
**notifier** — the side effect — is a double.

Feature tests extend `Tests\TestCase` (boots Laravel) and use the `RefreshDatabase` trait. Look at
`phpunit.xml` again:

```xml
<!-- phpunit.xml (excerpt — already there) -->
<env name="DB_CONNECTION" value="sqlite"/>
<env name="DB_DATABASE" value=":memory:"/>
<env name="MAIL_MAILER" value="array"/>
<env name="QUEUE_CONNECTION" value="sync"/>
```

**What happens:**

1. During tests, Laravel uses an **SQLite database in memory**, not your PostgreSQL. It is created
   fresh for the test run and disappears afterwards. No Docker needed, and your development data is
   never touched.
2. `RefreshDatabase` runs your migrations once at the start of the run, and wraps **every test in a
   transaction** that is rolled back at the end. Each test starts with empty tables.
3. `MAIL_MAILER=array` means even a real `Mail::send()` only stores the message in memory.

**The trade-off of SQLite.** It is fast and needs nothing installed except the `pdo_sqlite` extension
(Herd has it). But it is **not PostgreSQL**: some column types, `ILIKE`, JSON operators and some
constraint details behave differently. A test can pass on SQLite and the same code fail on PostgreSQL.
For today's tests (simple inserts and selects) that does not matter. In lesson 14 you will run the
feature tests against PostgreSQL to catch those differences.

Check that the feature setup works:

```bash
php artisan test --testsuite=Feature
```

**Check it works:** `Tests\Feature\ExampleTest` passes. If you see `could not find driver (Connection:
sqlite ...)`, enable `pdo_sqlite` in your `php.ini` (see Troubleshooting).

---

## Step 2 — The first mock: the warning is sent at 80 %

**Why:** this is the central behaviour of R9. We replace the `Notifier` with a mock that **expects**
exactly one call.

Choose numbers you can check in your head: rate 60 €/h is 1 € per minute; a cap of 100 € means **every
logged minute is exactly 1 % of the cap**. 80 minutes = 80 %.

```php
<?php

// tests/Feature/Services/BudgetMonitorTest.php

namespace Tests\Feature\Services;

use App\Contracts\Notifier;
use App\Enums\BillingType;
use App\Models\Client;
use App\Models\Project;
use App\Services\BudgetMonitor;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Carbon;
use Mockery;
use Mockery\MockInterface;
use Tests\TestCase;

class BudgetMonitorTest extends TestCase
{
    use RefreshDatabase;

    protected function setUp(): void
    {
        parent::setUp();

        $this->travelTo(Carbon::parse('2026-10-05 14:30:00'));
    }

    public function test_warns_the_owner_when_the_project_reaches_80_percent(): void
    {
        $project = $this->projectWithLoggedMinutes(80);

        $this->mock(Notifier::class, function (MockInterface $mock): void {
            $mock->shouldReceive('notify')
                ->once()
                ->with(
                    Mockery::on(fn (string $subject): bool => str_contains($subject, 'Website redesign')),
                    Mockery::type('string'),
                );
        });

        app(BudgetMonitor::class)->check($project);

        $this->assertSame(
            '2026-10-05 14:30:00',
            $project->fresh()->budget_warning_sent_at->toDateTimeString(),
        );
    }

    /**
     * A capped project: 60 €/h (1 € per minute) and a cap of 100 €,
     * so every logged minute is exactly 1 % of the cap.
     *
     * @param  array<string, mixed>  $overrides
     */
    private function projectWithLoggedMinutes(int $minutes, array $overrides = []): Project
    {
        $project = Project::factory()->for(Client::factory())->create([
            'name' => 'Website redesign',
            'billing_type' => BillingType::Capped,
            'hourly_rate_cents' => 6000,
            'budget_cap_cents' => 10000,
            'fixed_price_cents' => null,
            'rounding_minutes' => 1,
            'archived_at' => null,
            'budget_warning_sent_at' => null,
            ...$overrides,
        ]);

        $project->timeEntries()->create([
            'started_at' => '2026-10-05 08:00:00',
            'ended_at' => Carbon::parse('2026-10-05 08:00:00')->addMinutes($minutes),
            'description' => 'Design work',
            'billable' => true,
        ]);

        return $project;
    }
}
```

**What happens:**

1. `setUp()` runs before every test. `$this->travelTo(...)` **freezes the clock**: from now on, every
   `now()` in the application — including the one inside `BudgetMonitor` — returns 5 October 2026,
   14:30. Laravel resets the clock automatically after each test.
2. **Arrange, part 1 — the data.** `projectWithLoggedMinutes(80)` creates a client, a capped project and
   one 80-minute billable entry in the test database. The factories and `$project->timeEntries()->create()`
   go straight to the database, not through `TimeEntryService`, so **no event is fired** while we
   arrange. (`timeEntries()->create()` fills `project_id` from the relation, like `$client->projects()->create()`
   in lesson 05.) `...$overrides` (the spread
   operator on an array with string keys) lets a test change single attributes.
3. **Arrange, part 2 — the double.** `$this->mock(Notifier::class, ...)` creates a Mockery mock of the
   interface **and puts it into the service container** in place of the real binding from
   `AppServiceProvider`. Whoever asks the container for a `Notifier` from now on gets the mock.
4. `shouldReceive('notify')->once()` is an **expectation**: "`notify` must be called exactly one time".
   `->with(...)` describes the arguments with **matchers**: the first must be a string containing the
   project name (`Mockery::on()` takes a function that returns true or false), the second any string. We
   do not check the full message text — it may be reworded, and that should not break this test.
5. **Act.** `app(BudgetMonitor::class)` lets the container build the monitor. It sees the constructor
   `(Notifier, HourlyBilling, ProjectService)` and injects the **mock** plus the real
   `HourlyBilling` and `ProjectService`. Order matters: the mock must be registered **before** the
   monitor is built.
6. **Assert, part 1 — interaction.** Mockery checks the expectation when the test ends. If `notify` was
   called zero or two times, the test fails with a clear message. Laravel's `TestCase` closes Mockery
   for you and counts the expectation as an assertion.
7. **Assert, part 2 — state.** `$project->fresh()` reloads the row from the database, so we see what
   was really **saved**. That matters here: the claim is an `UPDATE` query, so the `$project` object in
   memory still says `null`. Because the clock is frozen, we know the exact value to expect.

We verify **both** the interaction (one notification) and the state (the timestamp). The timestamp is
what makes R9's "once" work, so it deserves its own assertion.

**Check it works:**

```bash
php artisan test --filter=BudgetMonitorTest
```

→ `✓ warns the owner when the project reaches 80 percent`.

**See it fail.** In `BudgetMonitor::check()`, temporarily comment out the `$this->notifier->notify(...)`
call and run again:

```text
  Mockery\Exception\InvalidCountException: Method notify(<Closure===true>, <string>)
  from Mockery_0_App_Contracts_Notifier should be called
   exactly 1 times but called 0 times.
```

Undo the change.

**Commit:** `test(budget): warning is sent at 80 % with a mocked notifier`

---

## Step 3 — Proving that nothing happened

**Why:** R9 says "once", and only for capped projects that reached 80 %. Each of these is a test that
something did **not** happen (README, section 4).

Add these methods to `BudgetMonitorTest`, above the private helper:

```php
// tests/Feature/Services/BudgetMonitorTest.php — add inside the class
    public function test_does_not_warn_at_79_percent(): void
    {
        $project = $this->projectWithLoggedMinutes(79);

        $this->mock(Notifier::class, function (MockInterface $mock): void {
            $mock->shouldReceive('notify')->never();
        });

        app(BudgetMonitor::class)->check($project);

        $this->assertNull($project->fresh()->budget_warning_sent_at);
    }

    public function test_does_not_warn_again_when_a_warning_was_already_sent(): void
    {
        $project = $this->projectWithLoggedMinutes(95, [
            'budget_warning_sent_at' => '2026-10-01 09:00:00',
        ]);

        $this->mock(Notifier::class, function (MockInterface $mock): void {
            $mock->shouldReceive('notify')->never();
        });

        app(BudgetMonitor::class)->check($project);

        $this->assertSame(
            '2026-10-01 09:00:00',
            $project->fresh()->budget_warning_sent_at->toDateTimeString(),
        );
    }

    public function test_does_not_warn_for_an_hourly_project(): void
    {
        $project = $this->projectWithLoggedMinutes(500, [
            'billing_type' => BillingType::Hourly,
            'budget_cap_cents' => null,
        ]);

        $this->mock(Notifier::class, function (MockInterface $mock): void {
            $mock->shouldReceive('notify')->never();
        });

        app(BudgetMonitor::class)->check($project);

        $this->assertNull($project->fresh()->budget_warning_sent_at);
    }

    public function test_a_second_check_does_not_send_a_second_warning(): void
    {
        $project = $this->projectWithLoggedMinutes(85);

        $this->mock(Notifier::class, function (MockInterface $mock): void {
            $mock->shouldReceive('notify')->once();
        });
        $monitor = app(BudgetMonitor::class);

        $monitor->check($project);
        $monitor->check($project->fresh());
    }

    public function test_does_not_warn_when_another_request_claimed_the_warning_first(): void
    {
        $project = $this->projectWithLoggedMinutes(85);

        // Another request sets the timestamp after we loaded $project; our copy still says null.
        Project::query()
            ->whereKey($project->id)
            ->update(['budget_warning_sent_at' => '2026-10-05 14:29:59']);

        $this->mock(Notifier::class, function (MockInterface $mock): void {
            $mock->shouldReceive('notify')->never();
        });

        app(BudgetMonitor::class)->check($project);

        $this->assertSame(
            '2026-10-05 14:29:59',
            $project->fresh()->budget_warning_sent_at->toDateTimeString(),
        );
    }

    public function test_releases_the_claim_when_the_notifier_fails(): void
    {
        $project = $this->projectWithLoggedMinutes(85);

        $this->mock(Notifier::class, function (MockInterface $mock): void {
            $mock->shouldReceive('notify')->once()->andThrow(new RuntimeException('Mail server down'));
        });

        $this->assertThrows(
            fn () => app(BudgetMonitor::class)->check($project),
            RuntimeException::class,
        );

        $this->assertNull($project->fresh()->budget_warning_sent_at);
    }
```

Add `use RuntimeException;` to the imports at the top.

**What happens:**

1. `shouldReceive('notify')->never()` is the "did not happen" expectation. If `notify` is called even
   once, the test fails immediately with `should be called exactly 0 times but called 1 times`.
2. The 79 % test and the 80 % test from step 2 are the **boundary pair** of R9. Together they prove the
   comparison is `>=` and not `>` (80 % must warn) and not something looser (79 % must not).
3. In the "already sent" test the old timestamp must be **unchanged** afterwards. A buggy version that
   skips the email but still overwrites the timestamp would be caught here.
4. The hourly project has 500 minutes (500 € of work) and no cap. `BudgetMonitor` must return before it
   even looks at the numbers.
5. The last test is the realistic "once" scenario: two entries are logged one after the other, so
   `check()` runs twice. `$project->fresh()` simulates what the listener does on the second event — it
   gets a freshly loaded project, which now has the timestamp. `->once()` fails if the second check sends
   again. This test has no explicit `assert...` line; the mock expectation **is** the assertion.
6. The "another request" test plays the race from lesson 09 in slow motion. We load the project, then
   change the row behind its back with a query-builder `update()` — exactly what a second request's
   claim would do. Our `$project` still says `null`, so the quick check at the top of `check()` lets it
   through. Only the claim's `WHERE budget_warning_sent_at IS NULL` stops it: 0 rows changed, no email,
   and the other request's timestamp stays.
7. The last test checks the retry. `andThrow(...)` makes the mock throw when `notify` is called, like a
   mail server that is down. `assertThrows` proves the exception still reaches the caller (the listener
   reports it), and the timestamp is `null` again, so the next logged entry can try again.

**Check it works:** `php artisan test --filter=BudgetMonitorTest` → 7 passed.

**See one fail.** In `BudgetMonitor::claimWarning()`, comment out the `->whereNull('budget_warning_sent_at')`
line. One test goes red: "does not warn when another request claimed the warning first". The quick
check at the top of `check()` still covers the other cases — only the claim protects against two
requests at the same moment. Undo the change. Then comment out the `releaseWarning($project)` call:
"releases the claim when the notifier fails" goes red, because the owner would never be warned after
one failed email. Undo that too.

**Commit:** `test(budget): no warning below 80 %, when already sent or claimed, or for hourly projects`

---

## Step 4 — The same check with a spy

**Why:** a mock sets its expectations **before** the act. Many people find it easier to read a test in
the order Arrange → Act → Assert, with the check at the end. A **spy** allows exactly that (README,
section 6).

Add to `BudgetMonitorTest`:

```php
// tests/Feature/Services/BudgetMonitorTest.php — add inside the class
    public function test_the_spy_version_reads_arrange_act_assert(): void
    {
        $project = $this->projectWithLoggedMinutes(80);
        $notifier = $this->spy(Notifier::class);

        app(BudgetMonitor::class)->check($project);

        $notifier->shouldHaveReceived('notify')->once();
    }
```

**What happens:**

1. `$this->spy(Notifier::class)` creates a Mockery **spy** and binds it in the container, like
   `$this->mock()`. A spy accepts **any** call and records it; it never fails during the act.
2. `shouldHaveReceived('notify')->once()` checks the record **after** the act.
3. Same behaviour tested as in step 2. Which style to use is a team decision; being consistent matters
   more. For "never called", the mock style (`->never()`) is often clearer because it fails at the exact
   line where the unwanted call happens. For the rest of this course we mostly use mocks.

---

## Step 5 — A hand-written fake: `InMemoryNotifier`

**Why:** a fake is a simple working implementation (README, section 10). It lets you inspect *everything*
that was sent with ordinary assertions, and it can be reused in many tests.

Create the folder `tests/Fakes`:

```php
<?php

// tests/Fakes/InMemoryNotifier.php

namespace Tests\Fakes;

use App\Contracts\Notifier;

/**
 * Test fake: remembers every notification instead of sending it.
 */
final class InMemoryNotifier implements Notifier
{
    /** @var list<array{subject: string, message: string}> */
    public array $sent = [];

    public function notify(string $subject, string $message): void
    {
        $this->sent[] = ['subject' => $subject, 'message' => $message];
    }
}
```

Add a test that uses it to `BudgetMonitorTest` (and `use Tests\Fakes\InMemoryNotifier;` at the top):

```php
// tests/Feature/Services/BudgetMonitorTest.php — add inside the class
    public function test_sends_exactly_one_warning_that_names_the_project(): void
    {
        $notifier = new InMemoryNotifier;
        $this->app->instance(Notifier::class, $notifier);
        $project = $this->projectWithLoggedMinutes(85);
        $monitor = app(BudgetMonitor::class);

        $monitor->check($project);
        $monitor->check($project->fresh());

        $this->assertCount(1, $notifier->sent);
        $this->assertSame('Budget warning: Website redesign', $notifier->sent[0]['subject']);
        $this->assertStringContainsString('85 %', $notifier->sent[0]['message']);
    }
```

**What happens:**

1. The fake lives in `tests/`, so it is never part of the application. The `Tests\` namespace is
   autoloaded from `tests/` (`autoload-dev` in `composer.json`).
2. `$this->app->instance(Notifier::class, $notifier)` puts **our object** into the container. It is
   what `$this->mock()` does internally — but here we keep a reference to a real object.
3. After the act, the assertions are plain PHPUnit: count the messages, read the subject. If the count
   is wrong, you can `dump($notifier->sent)` and see exactly what was sent.
4. `'85 %'`: `BudgetMonitor` writes the percentage into the message (`intdiv(value × 100, cap)` = 85).
   Checking a small, meaningful part of the text is fine; checking the whole sentence would make the test
   brittle.

**Compare the two versions** of "only one warning":

| | Mock (step 3) | Fake (this step) |
|---|---|---|
| Where is the check? | In the expectation, before the act | In normal assertions, after the act |
| What can you check? | What the matchers describe | Anything: count, order, full content |
| Failure message | "should be called exactly 1 times but called 2 times" | "Failed asserting that actual size 2 matches expected size 1" — and you can dump the list |
| Reuse | Written again in each test | One class for every test that needs a notifier |
| Best for | "Never called", one precise interaction | Scenarios with several calls; inspecting content |

**Check it works:** `php artisan test --filter=BudgetMonitorTest` → 9 passed.

**Commit:** `test(budget): add InMemoryNotifier fake`

---

## Step 6 — `Event::fake()`: `TimeEntryService` dispatches `TimeEntryLogged`

**Why:** `TimeEntryService` is the **publisher** in the Observer pattern (lesson 09). Its job is to
announce "an entry was logged"; what listeners do is not its business. So we test only the
announcement — with the listener switched off.

```php
<?php

// tests/Feature/Services/TimeEntryServiceTest.php

namespace Tests\Feature\Services;

use App\Events\TimeEntryLogged;
use App\Exceptions\ProjectArchivedException;
use App\Models\Client;
use App\Models\Project;
use App\Services\TimeEntryService;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Carbon;
use Illuminate\Support\Facades\Event;
use Tests\TestCase;

class TimeEntryServiceTest extends TestCase
{
    use RefreshDatabase;

    protected function setUp(): void
    {
        parent::setUp();

        Event::fake([TimeEntryLogged::class]);
        $this->travelTo(Carbon::parse('2026-10-05 18:00:00'));
    }

    public function test_logging_an_entry_dispatches_time_entry_logged(): void
    {
        $project = $this->activeProject();

        $entry = app(TimeEntryService::class)->log($this->validData($project));

        Event::assertDispatched(
            TimeEntryLogged::class,
            fn (TimeEntryLogged $event): bool => $event->timeEntry->is($entry),
        );
        Event::assertDispatchedTimes(TimeEntryLogged::class, 1);
    }

    public function test_an_entry_on_an_archived_project_is_rejected_and_nothing_is_dispatched(): void
    {
        $project = $this->activeProject(['archived_at' => '2026-09-30 12:00:00']);

        $this->assertThrows(
            fn () => app(TimeEntryService::class)->log($this->validData($project)),
            ProjectArchivedException::class,
        );

        Event::assertNotDispatched(TimeEntryLogged::class);
        $this->assertDatabaseCount('time_entries', 0);
    }

    /**
     * @param  array<string, mixed>  $overrides
     */
    private function activeProject(array $overrides = []): Project
    {
        return Project::factory()->for(Client::factory())->create([
            'archived_at' => null,
            ...$overrides,
        ]);
    }

    /**
     * The validated form data that TimeEntryController passes to log().
     *
     * @param  array<string, mixed>  $overrides
     * @return array<string, mixed>
     */
    private function validData(Project $project, array $overrides = []): array
    {
        return [
            'project_id' => $project->id,
            'started_at' => '2026-10-05 09:00',
            'ended_at' => '2026-10-05 10:30',
            'description' => 'Wireframes for the start page',
            'billable' => true,
            'tags' => [],
            ...$overrides,
        ];
    }
}
```

**What happens:**

1. `Event::fake([TimeEntryLogged::class])` replaces the event dispatcher with a **fake** — but only for
   this one event class. `TimeEntryLogged::dispatch(...)` is now recorded instead of delivered, so
   `CheckProjectBudget` and `BudgetMonitor` do not run. All other events (including Eloquent's own model
   events) still work normally. Plain `Event::fake()` with no list would fake *everything*, which can
   break model features that rely on events.
2. The clock is frozen at 18:00 so the "no entries in the future" rule does not depend on the real time:
   our entry ends at 10:30 the same day, which is always in the past.
3. `Event::assertDispatched(Class, closure)` passes if at least one recorded event makes the closure
   return `true`. The closure is our **captor**: it receives the real event object, and we check that it
   carries the entry that `log()` returned. `$model->is($other)` compares two models by table and key.
4. `assertDispatchedTimes(..., 1)` adds "exactly once".
5. `$this->assertThrows(closure, ExceptionClass)` runs the closure and passes only if it throws that
   exception. Unlike `expectException()`, the test **continues** afterwards — so we can still check the
   side effects: **no event** and **no row** in `time_entries`.
6. `validData()` mirrors what `StoreTimeEntryRequest::validated()` gives the service in production. The
   service does not know it is called from a test.

**Check it works:**

```bash
php artisan test --filter=TimeEntryServiceTest
```

→ 2 passed. The remaining rules (future entry, unknown project) are your Tasks 1 and 2.

**Commit:** `test(time-entries): TimeEntryLogged is dispatched only for accepted entries`

---

## Step 7 — `Mail::fake()`: testing the adapter itself

**Why:** we mock `Notifier` in the `BudgetMonitor` tests because we **own** it. The adapter
`MailNotifier` talks to Laravel's mailer, which we do **not** own — so we do not mock the `Mail` facade
with Mockery. We use the **framework fake** `Mail::fake()` instead (README, sections 6 and 8).

There is one problem. Lesson 09's `MailNotifier` uses `Mail::raw()`. The mail fake **does not record raw
messages** — `Mail::raw()` does nothing under `Mail::fake()`, and `Mail::assertSent()` has nothing to find.
The fake works with **mailable** classes. So the test tells us to change the design slightly. That is
a small refactor with a real benefit: the email becomes a class that can be tested and previewed.

### 7a — A mailable for notifications

```bash
php artisan make:mail NotificationMail
```

Replace the generated file:

```php
<?php

// app/Mail/NotificationMail.php

namespace App\Mail;

use Illuminate\Mail\Mailable;
use Illuminate\Mail\Mailables\Content;
use Illuminate\Mail\Mailables\Envelope;

/**
 * A plain-text email that carries one Notifier message to the owner.
 */
class NotificationMail extends Mailable
{
    public function __construct(
        public readonly string $subjectLine,
        public readonly string $messageText,
    ) {}

    public function envelope(): Envelope
    {
        return new Envelope(subject: $this->subjectLine);
    }

    public function content(): Content
    {
        return new Content(text: 'mail.notification');
    }
}
```

```blade
{{-- resources/views/mail/notification.blade.php --}}
{!! $messageText !!}
```

**What happens:**

1. A **mailable** is a class that describes one email: `envelope()` gives the subject, `content()` the
   view. Public properties are available in the view.
2. The properties are called `subjectLine` and `messageText` because `Mailable` already has a public
   `$subject` property, and mail views always receive a `$message` variable from Laravel. Other names
   avoid both clashes.
3. `Content(text: ...)` makes a **plain-text** email, the same as `Mail::raw()` before. In a text view we
   print with `{!! !!}` (unescaped): `{{ }}` would turn `"` into `&quot;`, which is correct for HTML but
   shows up literally in a text email. There is no HTML to inject into here.

### 7b — `MailNotifier` sends the mailable

```php
<?php

// app/Notifiers/MailNotifier.php

namespace App\Notifiers;

use App\Contracts\Notifier;
use App\Mail\NotificationMail;
use Illuminate\Container\Attributes\Config;
use Illuminate\Support\Facades\Mail;

/**
 * Adapter: turns Billable's notify(subject, message) into an email to the owner.
 */
final class MailNotifier implements Notifier
{
    public function __construct(
        #[Config('billable.owner_email')] private string $ownerEmail,
    ) {}

    public function notify(string $subject, string $message): void
    {
        Mail::to($this->ownerEmail)->send(new NotificationMail($subject, $message));
    }
}
```

`BudgetMonitor` does not change — it still calls `notify(subject, message)`. That is the Adapter
pattern paying off: the adapter's inside changed, its interface did not. The constructor with
`#[Config('billable.owner_email')]` stays exactly as in lesson 09, so the binding in
`AppServiceProvider` does not change either.

### 7c — The test

```php
<?php

// tests/Feature/Notifiers/MailNotifierTest.php

namespace Tests\Feature\Notifiers;

use App\Mail\NotificationMail;
use App\Notifiers\MailNotifier;
use Illuminate\Support\Facades\Mail;
use Tests\TestCase;

class MailNotifierTest extends TestCase
{
    public function test_sends_the_notification_as_one_email_to_the_owner(): void
    {
        Mail::fake();
        $notifier = new MailNotifier('owner@example.test');

        $notifier->notify('Budget warning: Website redesign', 'Project "Website redesign" has reached 80 %.');

        Mail::assertSent(NotificationMail::class, function (NotificationMail $mail): bool {
            return $mail->hasTo('owner@example.test')
                && $mail->hasSubject('Budget warning: Website redesign')
                && $mail->messageText === 'Project "Website redesign" has reached 80 %.';
        });
        Mail::assertSentCount(1);
    }
}
```

**What happens:**

1. `Mail::fake()` swaps Laravel's mailer for `MailFake`. Nothing is sent, not even to Mailpit; every
   mailable passed to `send()` is recorded.
2. We create `MailNotifier` with `new` and pass a test address. The `#[Config]` attribute is only read
   when the **container** builds the class (as the binding in `AppServiceProvider` does with
   `$app->make()`); with `new`, the argument we pass is used and the config is not involved. So the test
   does not depend on `.env`.
3. `Mail::assertSent(NotificationMail::class, closure)` finds recorded mailables of that class and passes
   if the closure returns `true` for one of them. `hasTo()` and `hasSubject()` are helpers on every
   mailable. Here we **do** check the exact text, because passing the message through unchanged is
   exactly this adapter's job.
4. The test needs Laravel (the facade and the fake), so it extends `Tests\TestCase`. It needs no
   database, so there is no `RefreshDatabase`.

**Check it works:**

```bash
php artisan test --filter=MailNotifierTest
```

→ 1 passed. Then check that the real thing still works: in Tinker, run
`app(App\Contracts\Notifier::class)->notify('Hello', 'Mailable version works.')` with
`BILLABLE_NOTIFIER=mail` and open Mailpit at `http://localhost:8025`. The email looks the same as before.

**Commit:** `refactor(notification): send notifications as a mailable and test with Mail::fake()`

---

## Step 8 — Run everything

```bash
./vendor/bin/pint
php artisan test
```

**Check it works:**

```text
   PASS  Tests\Unit\Support\MoneyTest
   ...
   PASS  Tests\Feature\Notifiers\MailNotifierTest
  ✓ sends the notification as one email to the owner
   PASS  Tests\Feature\Services\BudgetMonitorTest
  ✓ warns the owner when the project reaches 80 percent
  ✓ does not warn at 79 percent
  ✓ does not warn again when a warning was already sent
  ✓ does not warn for an hourly project
  ✓ a second check does not send a second warning
  ✓ does not warn when another request claimed the warning first
  ✓ releases the claim when the notifier fails
  ✓ the spy version reads arrange act assert
  ✓ sends exactly one warning that names the project
   PASS  Tests\Feature\Services\TimeEntryServiceTest
  ✓ logging an entry dispatches time entry logged
  ✓ an entry on an archived project is rejected and nothing is dispatched
```

Push. The CI `tests` job from lesson 11 runs the feature tests too — on SQLite in memory, so it needs no
database service.

**Commit:** `test: budget, time entry and notifier tests with doubles`

---

## Troubleshooting

| Error or symptom | Cause | Fix |
|---|---|---|
| `could not find driver (Connection: sqlite, ...)` | `pdo_sqlite` extension not enabled | Enable `extension=pdo_sqlite` in `php.ini` (`php --ini` shows which file); Herd has it enabled |
| `Method notify(...) should be called exactly 1 times but called 0 times` although the code works in the browser | The monitor was created **before** `$this->mock()`, so it holds the real notifier | Register the mock first, then `app(BudgetMonitor::class)` |
| Mock is ignored, real email appears in Mailpit during tests | Same cause, or the class under test was built with `new ...(new MailNotifier(...))` | Let the container build it after the mock is bound |
| `Mockery\Exception\NoMatchingExpectationException: No matching handler found for ...notify(...)` | Arguments do not match `->with(...)` (e.g. the subject has a different project name) | Check the matcher; print the real arguments with a spy |
| `Mockery\Exception\BadMethodCallException: Method ... does not exist on this mock object` | `shouldReceive` with a misspelt method name, or mocking a class instead of the interface | Use the exact method name from `Notifier` |
| `The class \App\Support\Money is marked final and its methods cannot be replaced` | Trying to mock a `final` class, e.g. the `Money` value object | Do not mock it — it is pure; use the real object (`Money::fromCents(500)`). Mock only collaborators such as `Notifier`; services stay non-final (lesson 07) |
| `Mail::assertSent` fails: `The expected [App\Mail\NotificationMail] mailable was not sent.` | `MailNotifier` still uses `Mail::raw()`, which the fake does not record | Step 7: send a mailable |
| `Event::assertDispatched` fails although `log()` dispatches | `Event::fake()` was called after the service was resolved, or the wrong event class is faked | Call `Event::fake([...])` in `setUp()`, before the act |
| Budget test: `budget_warning_sent_at` differs by hours | App timezone and the string in the test differ | `config/app.php` timezone must be `Europe/Tallinn` (lesson 01); compare with `toDateTimeString()` |
| `Call to a member function toDateTimeString() on null` | No warning was claimed, or `budget_warning_sent_at` is not cast to `datetime` | Check `casts()` on `Project`; check that `check()` calls `claimWarning()` and that `notify()` did not throw (a failure releases the claim) |
| "another request claimed the warning first" fails: `notify` called 1 times | The claim is not conditional — `whereNull('budget_warning_sent_at')` is missing, or the timestamp is set with `$project->save()` | Use the conditional `update()` from lesson 09 and act only when it returns `1` |
| `Illuminate\Database\QueryException ... NOT NULL constraint failed: projects.hourly_rate_cents` | Your factory or migration needs a value you set to `null` | Give the attribute a value in the override, or fix the factory state |
| `TimeEntryInFutureException` in `TimeEntryServiceTest` | Clock not frozen, test data ends later than "now" | `travelTo()` in `setUp()`; entries end before 18:00 |

---

## Recap

- New: `tests/Feature/Services/BudgetMonitorTest.php`, `tests/Feature/Services/TimeEntryServiceTest.php`,
  `tests/Feature/Notifiers/MailNotifierTest.php`, `tests/Fakes/InMemoryNotifier.php`,
  `app/Mail/NotificationMail.php`, `resources/views/mail/notification.blade.php`; changed
  `app/Notifiers/MailNotifier.php`.
- `$this->mock(Notifier::class, fn ...)` puts a Mockery mock into the container; `->once()` and
  `->never()` are expectations checked when the test ends.
- `$this->spy()` + `shouldHaveReceived()` is the same check written after the act.
- `$this->app->instance(Notifier::class, $fake)` binds a hand-written fake.
- `$this->travelTo()` freezes `now()` for the whole application during one test.
- `->andThrow(...)` makes a mock fail like a broken mail server; with `assertThrows` we check that the
  claim is released.
- Changing the row with a query-builder `update()` behind the loaded model's back simulates a second
  request, so the atomic claim can be tested without threads.
- `Event::fake([TimeEntryLogged::class])` records one event type and lets the rest run.
- `Mail::fake()` records mailables — not `Mail::raw()` — so the adapter now sends a mailable.
- Half the budget tests prove that something did **not** happen.

---

## Independent work — solutions

> **The tasks are not here.** The task descriptions (Basic / Intermediate / Advanced, with acceptance
> criteria) are in this lesson's [README.md → Independent work (~2 h)](README.md#independent-work-2-h).
> Read them there first and build the features in your project. This section only contains the
> solutions — try each task yourself, then open a solution to compare or when you are stuck.

<details>
<summary><strong>Task 1 (Basic)</strong> — tests for every <code>TimeEntryService</code> rule</summary>

In Laravel, `TimeEntryService` talks to Eloquent directly, so the realistic doubles are the **event fake**
and the **frozen clock**; the database is the real test database. Add to `TimeEntryServiceTest`:

```php
// tests/Feature/Services/TimeEntryServiceTest.php — add inside the class
// imports to add: App\Models\Tag, App\Models\TimeEntry,
//                 Illuminate\Database\Eloquent\ModelNotFoundException
    public function test_a_valid_entry_is_saved_with_its_tags(): void
    {
        $project = $this->activeProject();
        $tag = Tag::forceCreate(['name' => 'design']);

        $entry = app(TimeEntryService::class)->log($this->validData($project, ['tags' => [$tag->id]]));

        $this->assertDatabaseHas('time_entries', [
            'id' => $entry->id,
            'project_id' => $project->id,
            'description' => 'Wireframes for the start page',
        ]);
        $this->assertTrue($entry->tags()->whereKey($tag->id)->exists());
    }

    public function test_an_unknown_project_is_rejected_and_nothing_is_dispatched(): void
    {
        $this->assertThrows(
            fn () => app(TimeEntryService::class)->log([
                'project_id' => 999999,
                'started_at' => '2026-10-05 09:00',
                'ended_at' => '2026-10-05 10:30',
                'description' => 'Ghost work',
                'billable' => true,
                'tags' => [],
            ]),
            ModelNotFoundException::class,
        );

        Event::assertNotDispatched(TimeEntryLogged::class);
        $this->assertSame(0, TimeEntry::count());
    }

    public function test_the_event_carries_the_entry_of_the_right_project(): void
    {
        $project = $this->activeProject();
        $this->activeProject(); // a second project that must not appear in the event

        app(TimeEntryService::class)->log($this->validData($project));

        Event::assertDispatched(
            TimeEntryLogged::class,
            fn (TimeEntryLogged $event): bool => $event->timeEntry->project_id === $project->id,
        );
    }
```

Together with the two tests from class, every rule of `log()` is covered: saved + event, archived,
unknown project, and (Task 2) the future rule.

**Why not a Mockery mock of `TimeEntry` or `Project`?** Eloquent models are not boundaries; they are the
data. A mocked model needs a stub for every attribute (`getAttribute('archived_at')`, …) and proves
nothing about the real query. The rules R4 (end after start, max 12 h) are checked by
`StoreTimeEntryRequest`, not by the service — they are tested with a feature test of the form in
lesson 14.

**Commit:** `test(time-entries): cover all TimeEntryService rules`

</details>

<details>
<summary><strong>Task 2 (Basic)</strong> — the future-entry rule with a frozen clock</summary>

`setUp()` already freezes "now" at **2026-10-05 18:00:00**. Add a data-driven test for the accepted
boundary rows and one test for the rejected row:

```php
// tests/Feature/Services/TimeEntryServiceTest.php — add inside the class
// imports to add: App\Exceptions\TimeEntryInFutureException,
//                 PHPUnit\Framework\Attributes\DataProvider
    #[DataProvider('endTimesInThePast')]
    public function test_an_entry_that_ends_by_now_is_accepted(string $endedAt): void
    {
        $project = $this->activeProject();

        app(TimeEntryService::class)->log($this->validData($project, [
            'started_at' => '2026-10-05 17:00',
            'ended_at' => $endedAt,
        ]));

        $this->assertDatabaseCount('time_entries', 1);
        Event::assertDispatchedTimes(TimeEntryLogged::class, 1);
    }

    public static function endTimesInThePast(): array
    {
        return [
            'one minute ago' => ['2026-10-05 17:59'],
            'exactly now' => ['2026-10-05 18:00'],
        ];
    }

    public function test_an_entry_that_ends_one_minute_in_the_future_is_rejected(): void
    {
        $project = $this->activeProject();

        $this->assertThrows(
            fn () => app(TimeEntryService::class)->log($this->validData($project, [
                'started_at' => '2026-10-05 17:00',
                'ended_at' => '2026-10-05 18:01',
            ])),
            TimeEntryInFutureException::class,
        );

        $this->assertDatabaseCount('time_entries', 0);
        Event::assertNotDispatched(TimeEntryLogged::class);
    }
```

**What happens:** the service calls `Carbon::parse($data['ended_at'])->isAfter(now())`. `now()` returns
the frozen 18:00:00. 17:59 and 18:00 are not after it (equal is not "after"), so they are accepted; 18:01
is rejected. Without `travelTo()`, the test that uses "18:01 today" would pass or fail depending on when
it runs — and on 5 October 2026 at 18:00:30 it would even flip in the middle of a run.

Try it: remove the `travelTo()` line from `setUp()` and run. The accepted rows fail with
`TimeEntryInFutureException` if you run the test before 5 October 2026 18:00. Put it back.

**Commit:** `test(time-entries): future-entry rule at its boundaries with a frozen clock`

</details>

<details>
<summary><strong>Task 3 (Intermediate)</strong> — a feature test for creating a client</summary>

In Laravel the "controller test" is a feature test: it sends a real request through routing, the Form
Request, the controller and the service to the test database.

```php
<?php

// tests/Feature/Http/ClientControllerTest.php

namespace Tests\Feature\Http;

use App\Models\Client;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class ClientControllerTest extends TestCase
{
    use RefreshDatabase;

    public function test_a_valid_client_is_created_and_shown(): void
    {
        $response = $this->post(route('clients.store'), [
            'name' => 'Saar OÜ',
            'email' => 'info@saar.ee',
            'vat_number' => 'EE101234567',
        ]);

        $client = Client::query()->where('email', 'info@saar.ee')->firstOrFail();
        $response->assertRedirect(route('clients.show', $client));
        $response->assertSessionHas('status'); // the text differs between lesson 05 and lesson 07's ClientService version
        $this->assertSame('Saar OÜ', $client->name);
    }

    public function test_invalid_input_returns_to_the_form_with_errors(): void
    {
        $response = $this->from(route('clients.create'))->post(route('clients.store'), [
            'name' => '',
            'email' => 'not-an-email',
        ]);

        $response->assertRedirect(route('clients.create'));
        $response->assertSessionHasErrors(['name', 'email']);
        $this->assertDatabaseCount('clients', 0);
    }

    public function test_a_duplicate_email_is_rejected(): void
    {
        Client::factory()->create(['email' => 'info@saar.ee']);

        $response = $this->from(route('clients.create'))->post(route('clients.store'), [
            'name' => 'Another Saar',
            'email' => 'info@saar.ee',
        ]);

        $response->assertSessionHasErrors(['email']);
        $response->assertSessionDoesntHaveErrors(['name']);
        $this->assertDatabaseCount('clients', 1);
    }
}
```

**What happens:**

1. `$this->post(route(...), [...])` sends a POST like the browser form. CSRF protection is disabled
   automatically in tests.
2. On a validation error, Laravel redirects **back**. In a test there is no previous page, so
   `$this->from(route('clients.create'))` sets it — then `assertRedirect(route('clients.create'))` is
   meaningful.
3. `assertSessionHasErrors(['name', 'email'])` checks the error bag the form would show.
   `assertSessionDoesntHaveErrors(['name'])` proves the duplicate test fails for the right reason.
4. There is **no mock** here, on purpose. The behaviour under test is the whole chain (validation rules,
   including `Rule::unique`, and the redirect), and the database is fast and local. A mocked
   `ClientService` would test almost nothing. Compare with the Spring track, where `@WebMvcTest` mocks
   the service and tests only the web layer — both are valid, with different trade-offs.

**Commit:** `test(clients): feature tests for creating a client`

</details>

<details>
<summary><strong>Task 4 (Advanced)</strong> — refactor an over-mocked test into a fake-based one</summary>

**The over-mocked version** — for reading, do not add it to your project:

```php
// An over-mocked test — every collaborator and even the model are doubles
public function test_warns_at_80_percent(): void
{
    $project = Mockery::mock(Project::class)->makePartial();
    $project->shouldReceive('getAttribute')->with('billing_type')->andReturn(BillingType::Capped);
    $project->shouldReceive('getAttribute')->with('budget_warning_sent_at')->andReturn(null);
    $project->shouldReceive('getAttribute')->with('budget_cap_cents')->andReturn(10000);
    $project->shouldReceive('getAttribute')->with('hourly_rate_cents')->andReturn(6000);
    $project->shouldReceive('getAttribute')->with('name')->andReturn('Website redesign');
    $project->shouldReceive('getAttribute')->with('id')->andReturn(42);

    $projects = Mockery::mock(ProjectService::class);
    $projects->shouldReceive('billableMinutes')->once()->with($project)->andReturn(80);

    $notifier = Mockery::mock(Notifier::class);
    $notifier->shouldReceive('notify')->once()->with(
        'Budget warning: Website redesign',
        'Project "Website redesign" has reached 80 % of its budget cap: 80,00 € of 100,00 €.',
    );

    (new BudgetMonitor($notifier, new HourlyBilling, $projects))->check($project);
}
```

**The fake-based version** is `test_sends_exactly_one_warning_that_names_the_project` from step 5 (plus
the state assertion on `budget_warning_sent_at` from step 2).

Example text for `docs/testing.md`:

```markdown
<!-- docs/testing.md — add this section -->
## Mock or fake?

The first version of the budget warning test mocked everything: the `Project` model (six stubbed
attributes), `ProjectService::billableMinutes()` and the notifier, which expected the complete
message text. It tested the implementation, not the rule. Renaming the subject or loading the
minutes in a different way would break it, although R9 still worked. And it did not even pass: the
claim in `check()` is a real `UPDATE … WHERE id = 42`. Without a database the query crashes; with
the test database there is no row 42, so the claim changes 0 rows and no warning is sent. Making it
green would need yet another double for the query. A bug in `billableMinutes()` or in the claim could
never be found this way.

The new test uses real data in the test database (SQLite in memory), the real `ProjectService`,
and a hand-written `InMemoryNotifier` fake. It checks what the owner would notice: exactly one
warning, whose subject names the project, and a stored `budget_warning_sent_at`. It is shorter and
survives refactoring.

I kept a Mockery mock with `->never()` for the "no warning at 79 %" and "no warning for hourly
projects" tests: there the whole point is one interaction that must not happen, and `never()` fails
at exactly the line where the unwanted call happens.
```

**Commit:** `test(budget): replace an over-mocked test with a fake; document the choice`

</details>
