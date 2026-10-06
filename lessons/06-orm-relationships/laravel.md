# 06 — ORM: relationships and the N+1 problem · Laravel

[← Concepts](README.md) · Starting point: end of lesson 05 · Estimated time in class: 2 h

---

## What we build today

- Three migrations: `time_entries`, `tags`, `tag_time_entry`.
- Models `TimeEntry` and `Tag`, with `belongsTo`, `hasMany` and `belongsToMany` relations.
- A seeder with about 60 time entries over the last three weeks, some with tags.
- The time sheet page `/time-entries`: first written naively (about 150 queries), then **measured**,
  then made to fail loudly with `Model::shouldBeStrict()`, then fixed with eager loading (4 queries).
- A form to log a new time entry, validated by R4 (end after start, at most 12 hours).

**Final result:** the navigation has a "Time sheet" link. The page shows the latest 50 entries with date,
time, project, client, description, duration as `h:mm` and tags. Your log file shows the number of
queries for each request, and it stays the same no matter how many entries exist.

This builds on lesson 05: `Client` `hasMany` projects, `Project` `belongsTo` client, projects CRUD, and
the `euros()` helper in `app/helpers.php`.

---

## Step 1 — Migrations

Create three migration files. Run the commands **in this order**: the file name starts with a timestamp,
and the pivot table must be created after both tables it points to.

```bash
php artisan make:migration create_time_entries_table
php artisan make:migration create_tags_table
php artisan make:migration create_tag_time_entry_table
```

```php
<?php

// database/migrations/xxxx_xx_xx_xxxxxx_create_time_entries_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('time_entries', function (Blueprint $table) {
            $table->id();
            $table->foreignId('project_id')->constrained()->restrictOnDelete();
            $table->timestamp('started_at');
            $table->timestamp('ended_at');
            $table->string('description', 255);
            $table->boolean('billable')->default(true);
            $table->timestamp('created_at');
            $table->timestamp('updated_at');

            $table->index(['project_id', 'started_at']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('time_entries');
    }
};
```

```php
<?php

// database/migrations/xxxx_xx_xx_xxxxxx_create_tags_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('tags', function (Blueprint $table) {
            $table->id();
            $table->string('name', 40)->unique();
            $table->timestamp('created_at');
            $table->timestamp('updated_at');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('tags');
    }
};
```

```php
<?php

// database/migrations/xxxx_xx_xx_xxxxxx_create_tag_time_entry_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('tag_time_entry', function (Blueprint $table) {
            $table->foreignId('tag_id')->constrained()->cascadeOnDelete();
            $table->foreignId('time_entry_id')->constrained()->cascadeOnDelete();

            $table->primary(['tag_id', 'time_entry_id']);
            $table->index('time_entry_id');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('tag_time_entry');
    }
};
```

```bash
php artisan migrate
```

**What happens:**

1. `foreignId('project_id')` creates a `bigint` column. `->constrained()` adds a foreign key to the table
   guessed from the column name: `project_id` → `projects.id`.
2. `->restrictOnDelete()` writes `ON DELETE RESTRICT`: PostgreSQL refuses to delete a project that still
   has time entries. Time entries are billing evidence (README, concept 2).
3. `$table->timestamp(...)` in PostgreSQL is `timestamp(0) without time zone`. Laravel stores the times in
   the application's timezone (Europe/Tallinn, set in lesson 01). `created_at` and `updated_at` are
   written out as two `NOT NULL` columns, as in lesson 03 (not the nullable `timestamps()` shortcut).
   The pivot table has no timestamps, so `attach()` and `sync()` only write the two ids.
4. `$table->index(['project_id', 'started_at'])` is the composite index from PROJECT.md. PostgreSQL does
   not index foreign-key columns on its own, so without it every "entries of project X" query would scan
   the whole table.
5. In the pivot table there is no `id` column: `primary(['tag_id', 'time_entry_id'])` makes the pair the
   primary key, so the same tag cannot be attached twice to one entry. `cascadeOnDelete()` on both sides
   removes the link rows when an entry or a tag is deleted.
6. `index('time_entry_id')`: the primary key starts with `tag_id`, so it cannot help the query "tags of
   these entries" (`WHERE time_entry_id IN (...)`), which the time sheet runs on every page view.
7. The pivot table name `tag_time_entry` follows Eloquent's convention: both model names, singular,
   snake_case, in alphabetical order.

**Check it works:**

```bash
php artisan migrate:status
docker compose exec postgres psql -U billable -d billable -c '\d time_entries'
```

The last command lists the columns, the index `time_entries_project_id_started_at_index` and the foreign
key `time_entries_project_id_foreign`.

**Commit:** `feat(time-entries): add time_entries, tags and pivot tables`

---

## Step 2 — Models and relationships

```bash
php artisan make:model TimeEntry
php artisan make:model Tag
```

```php
<?php

// app/Models/TimeEntry.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class TimeEntry extends Model
{
    protected $fillable = [
        'project_id',
        'started_at',
        'ended_at',
        'description',
        'billable',
    ];

    protected function casts(): array
    {
        return [
            'started_at' => 'datetime',
            'ended_at' => 'datetime',
            'billable' => 'boolean',
        ];
    }

    /**
     * @return BelongsTo<Project, $this>
     */
    public function project(): BelongsTo
    {
        return $this->belongsTo(Project::class);
    }

    /**
     * @return BelongsToMany<Tag, $this>
     */
    public function tags(): BelongsToMany
    {
        return $this->belongsToMany(Tag::class);
    }

    /**
     * Length of the entry in whole minutes, not rounded (rounding rules come in lesson 10).
     */
    public function durationMinutes(): int
    {
        return (int) $this->started_at->diffInMinutes($this->ended_at);
    }
}
```

```php
<?php

// app/Models/Tag.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class Tag extends Model
{
    protected $fillable = ['name'];

    /**
     * @return BelongsToMany<TimeEntry, $this>
     */
    public function timeEntries(): BelongsToMany
    {
        return $this->belongsToMany(TimeEntry::class);
    }
}
```

Add the other side of the project relation to `app/Models/Project.php`. We also need the `active()`
scope from the lesson 03 independent work; if you skipped that task, add it now too:

```php
// app/Models/Project.php (add the imports at the top and the methods inside the class)
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Relations\HasMany;

/**
 * @return HasMany<TimeEntry, $this>
 */
public function timeEntries(): HasMany
{
    return $this->hasMany(TimeEntry::class);
}

/**
 * Only projects that are not archived (lesson 03 independent work — skip if you have it).
 *
 * @param  Builder<Project>  $query
 */
public function scopeActive(Builder $query): void
{
    $query->whereNull('archived_at');
}
```

**What happens:**

1. `belongsTo(Project::class)` means "this table has the foreign key". Eloquent guesses the column from
   the method name: `project` → `project_id`.
2. `hasMany(TimeEntry::class)` is the other direction: "the **other** table has a `project_id` pointing
   at me".
3. `belongsToMany(Tag::class)` means "linked through a pivot table". Eloquent guesses the table
   (`tag_time_entry`) and both columns (`time_entry_id`, `tag_id`). Both `TimeEntry::tags()` and
   `Tag::timeEntries()` use the same pivot table — Eloquent has no "owning side".
4. A relation **method** (`$entry->tags()`) returns a query builder you can extend or use to attach
   tags. A relation **property** (`$entry->tags`) runs the query (if not loaded yet) and returns the
   result. Remember this difference — N+1 hides in the property.
5. The `datetime` casts turn the database strings into Carbon objects, so `$entry->started_at->format(...)`
   works.
6. `diffInMinutes()` — **version pitfall**: since Carbon 3 (Laravel 11 and newer) it returns a `float`
   and is **signed** (negative if the second date is earlier). Old tutorials assume an `int` and an
   absolute value. We call it from start to end and cast to `int`.
7. `scopeActive` is a **local scope**. The `scope` prefix is removed and the first letter lower-cased:
   `Project::active()`. (Laravel 12 also offers a `#[Scope]` attribute; the `scope...` prefix works in all
   versions.) `isArchived()` (lesson 03) answers the question for one project in PHP; the scope is the
   query version of the same rule, answered by the database.

**Check it works:**

```bash
php artisan tinker
```

```php
> App\Models\Project::active()->count()
= 6
> App\Models\Project::first()->timeEntries()->toRawSql()
= "select * from "time_entries" where "time_entries"."project_id" = 1 and ..."
```

(`toRawSql()` exists in Laravel 10.15 and newer. The exact SQL may differ slightly.)

---

## Step 3 — Seed 60 time entries

```bash
php artisan make:seeder TimeEntrySeeder
```

```php
<?php

// database/seeders/TimeEntrySeeder.php

namespace Database\Seeders;

use App\Models\Project;
use App\Models\Tag;
use App\Models\TimeEntry;
use Illuminate\Database\Seeder;

class TimeEntrySeeder extends Seeder
{
    private const array TAGS = ['meeting', 'bugfix', 'design', 'review', 'support'];

    private const array DESCRIPTIONS = [
        'Kick-off call',
        'Fix login bug',
        'Code review',
        'Design home page',
        'Write API documentation',
        'Weekly meeting',
        'Deploy to staging',
        'Answer support tickets',
    ];

    /** Possible entry lengths in minutes. */
    private const array LENGTHS = [30, 45, 60, 90, 120];

    public function run(): void
    {
        $projects = Project::active()->get();

        if ($projects->isEmpty()) {
            $this->command->warn('No active projects. Run the client and project seeders first.');

            return;
        }

        $tags = collect(self::TAGS)->map(fn (string $name) => Tag::firstOrCreate(['name' => $name]));

        mt_srand(42); // same "random" data every time

        $monday = today()->subWeeks(3)->startOfWeek();

        // 15 working days × 4 entries = 60 entries
        for ($day = 0; $day < 15; $day++) {
            $start = $monday->copy()->addWeekdays($day)->setTime(9, 0);

            for ($slot = 0; $slot < 4; $slot++) {
                $minutes = self::LENGTHS[mt_rand(0, count(self::LENGTHS) - 1)];

                $entry = TimeEntry::create([
                    'project_id' => $projects[mt_rand(0, $projects->count() - 1)]->id,
                    'started_at' => $start,
                    'ended_at' => $start->copy()->addMinutes($minutes),
                    'description' => self::DESCRIPTIONS[mt_rand(0, count(self::DESCRIPTIONS) - 1)],
                    'billable' => mt_rand(1, 6) !== 1, // about 1 in 6 is not billable
                ]);

                $entry->tags()->attach($tags->random(mt_rand(0, 2))->pluck('id'));

                $start = $entry->ended_at->copy()->addMinutes(15); // 15 min break
            }
        }
    }
}
```

Register it in `database/seeders/DatabaseSeeder.php`. Keep the 3 clients and 7 projects from lesson 03
exactly as they are and add one line at the **end** of `run()`, so the projects exist when the new seeder
runs:

```php
// database/seeders/DatabaseSeeder.php (last line inside run())
$this->call(TimeEntrySeeder::class);
```

Your database already has the clients and projects, so run only the new seeder now:

```bash
php artisan db:seed --class=TimeEntrySeeder
```

(Later you can rebuild everything from zero with `php artisan migrate:fresh --seed`. That **deletes all
data** in the database.)

**What happens:**

1. `Tag::firstOrCreate(['name' => ...])` finds the tag or creates it. Running the seeder twice does not
   create duplicate tags (and would not be allowed by the `UNIQUE` constraint anyway).
2. `today()->subWeeks(3)->startOfWeek()` is Monday 00:00 three weeks ago. `addWeekdays($day)` skips
   Saturdays and Sundays, so 15 working days are exactly the last three full weeks.
3. `copy()` matters: `Illuminate\Support\Carbon` is **mutable**. `$start->addMinutes(90)` would change
   `$start` itself.
4. `mt_srand(42)` fixes the random generator, so everyone in class gets the same data and the same query
   counts.
5. `$entry->tags()->attach([...])` inserts rows into `tag_time_entry`. `$tags->random(0)` returns an
   empty collection, so some entries have no tags.

**Check it works:**

```bash
php artisan tinker --execute="echo App\Models\TimeEntry::count(), ' entries, ', DB::table('tag_time_entry')->count(), ' tag links';"
```

You see `60 entries, ` followed by roughly 60 tag links.

**Commit:** `feat(time-entries): seed time entries and tags`

---

## Step 4 — The naive time sheet

First a small helper for durations. Add it to `app/helpers.php`, next to `euros()`:

```php
// app/helpers.php (add below the other helpers)
if (! function_exists('hours_minutes')) {
    /**
     * 90 → "1:30", 5 → "0:05".
     */
    function hours_minutes(int $minutes): string
    {
        return sprintf('%d:%02d', intdiv($minutes, 60), $minutes % 60);
    }
}
```

Routes — add to `routes/web.php`:

```php
// routes/web.php
use App\Http\Controllers\TimeEntryController;

Route::resource('time-entries', TimeEntryController::class)->only(['index', 'create', 'store']);
```

```bash
php artisan make:controller TimeEntryController
```

```php
<?php

// app/Http/Controllers/TimeEntryController.php

namespace App\Http\Controllers;

use App\Models\TimeEntry;
use Illuminate\View\View;

class TimeEntryController extends Controller
{
    public function index(): View
    {
        // NAIVE VERSION — we measure it in step 5 and fix it in step 7.
        $entries = TimeEntry::query()
            ->orderByDesc('started_at')
            ->limit(50)
            ->get();

        return view('time-entries.index', ['entries' => $entries]);
    }
}
```

```blade
{{-- resources/views/time-entries/index.blade.php --}}
@extends('layouts.app')

@section('title', 'Time sheet')

@section('content')
    <h1>Time sheet</h1>

    <p><a href="{{ route('time-entries.create') }}" role="button">Log time</a></p>

    <div class="overflow-auto">
        <table>
            <thead>
                <tr>
                    <th>Date</th>
                    <th>Time</th>
                    <th>Project</th>
                    <th>Client</th>
                    <th>Description</th>
                    <th>Duration</th>
                    <th>Tags</th>
                </tr>
            </thead>
            <tbody>
                @forelse ($entries as $entry)
                    <tr>
                        <td>{{ $entry->started_at->format('d.m.Y') }}</td>
                        <td>{{ $entry->started_at->format('H:i') }}–{{ $entry->ended_at->format('H:i') }}</td>
                        <td><a href="{{ route('projects.show', $entry->project) }}">{{ $entry->project->name }}</a></td>
                        <td>{{ $entry->project->client->name }}</td>
                        <td>
                            {{ $entry->description }}
                            @unless ($entry->billable)
                                <small>(not billable)</small>
                            @endunless
                        </td>
                        <td>{{ hours_minutes($entry->durationMinutes()) }}</td>
                        <td>
                            @foreach ($entry->tags as $tag)
                                <mark>{{ $tag->name }}</mark>
                            @endforeach
                        </td>
                    </tr>
                @empty
                    <tr>
                        <td colspan="7">No time entries yet.</td>
                    </tr>
                @endforelse
            </tbody>
        </table>
    </div>
@endsection
```

Add a link to the navigation in `resources/views/layouts/app.blade.php`, next to "Clients":

```blade
{{-- resources/views/layouts/app.blade.php (inside the second <ul> of the nav, after Clients) --}}
<li>
    <a href="{{ route('time-entries.index') }}" @if (request()->routeIs('time-entries.*')) aria-current="page" @endif>Time sheet</a>
</li>
```

We have not written `create()` yet. The "Log time" link needs only the route name, which exists, so the
page works; clicking the link gives an error until step 8.

**What happens:**

1. `orderByDesc('started_at')->limit(50)->get()` runs **one** query and returns 50 `TimeEntry` models.
   Their relations are **not** loaded.
2. In the view, `$entry->project` (property, no parentheses) sees that `project` is not loaded and runs
   `select * from projects where id = ?` — for **this** entry. The next entry does the same again, even
   if it has the same project: Eloquent has no identity map.
3. `$entry->project->client` does the same for the client of each project object.
4. `$entry->tags` runs one pivot query per entry.
5. `overflow-auto` is a Pico.css class: on a phone the table scrolls sideways instead of breaking the page.

**Check it works:** open `/time-entries`. The page looks correct: 50 rows, newest first. Nothing
looks wrong — and that is exactly the danger.

---

## Step 5 — Measure: count the queries per request

We want the number of queries for every request in the log. The view is rendered **after** the
controller method returns, so counting inside the controller would miss the view's queries. Instead we
listen for every query and write the total when the request is finished.

Open `app/Providers/AppServiceProvider.php` and change it to this (keep anything else you already have in
`register()` or `boot()`):

```php
<?php

// app/Providers/AppServiceProvider.php

namespace App\Providers;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Pagination\Paginator;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        //
    }

    public function boot(): void
    {
        Model::preventSilentlyDiscardingAttributes(! $this->app->isProduction()); // lesson 03
        Paginator::defaultView('partials.pagination'); // lesson 04

        if ($this->app->isLocal() && ! $this->app->runningInConsole()) {
            $this->logQueryCountPerRequest();
        }
    }

    /**
     * Development helper: writes "GET /time-entries: 4 queries" to the log for every request.
     */
    private function logQueryCountPerRequest(): void
    {
        $count = 0;

        DB::listen(function () use (&$count): void {
            $count++;
        });

        $this->app->terminating(function () use (&$count): void {
            $request = request();
            Log::debug("{$request->method()} /{$request->path()}: {$count} queries");
        });
    }
}
```

Watch the log in a second terminal:

```bash
tail -f storage/logs/laravel.log
```

Reload `/time-entries`.

**What happens:**

1. `DB::listen()` registers a function that Laravel calls after **every** SQL query. We only count.
   (The function also receives a `QueryExecuted` object with `sql`, `bindings` and `time` — useful when you
   want to see the queries themselves.)
2. `use (&$count)` passes the variable **by reference**, so both closures share the same counter.
3. `terminating()` runs a function at the very end of the request, after the response (and the view)
   was produced.
4. `isLocal()` limits this to `APP_ENV=local`. `runningInConsole()` skips Artisan commands and tests.

**Check it works:** the log shows a line like

```
local.DEBUG: GET /time-entries: 153 queries
```

151 queries come from the page: 1 for the entries, 50 for projects, 50 for clients, 50 for tags. The
remaining 1–2 come from the session (Laravel stores sessions in the database by default,
`SESSION_DRIVER=database`). Write your number down.

To see the queries themselves, temporarily log `$query->sql` inside the listener
(`function (QueryExecuted $query) use (&$count)`, import `Illuminate\Database\Events\QueryExecuted`). You
will see the same `select * from "projects" where "projects"."id" = ? limit 1` many times.

---

## Step 6 — Make lazy loading fail loudly

Counting works, but only if somebody looks at the log. We want the mistake to be **impossible to miss** in
development. Replace the lesson 03 guard at the top of `boot()` with `Model::shouldBeStrict()`:

```php
// app/Providers/AppServiceProvider.php (boot, top)

public function boot(): void
{
    Model::shouldBeStrict(! $this->app->isProduction()); // replaces the lesson 03 line
    Paginator::defaultView('partials.pagination'); // lesson 04

    if ($this->app->isLocal() && ! $this->app->runningInConsole()) {
        $this->logQueryCountPerRequest();
    }
}
```

Reload `/time-entries`.

**What you see:**

```
Illuminate\Database\LazyLoadingViolationException
Attempted to lazy load [project] on model [App\Models\TimeEntry] but lazy loading is disabled.
```

**What happens:**

1. `shouldBeStrict(true)` switches on three guards at once. Each one turns a silent mistake into an
   exception:
   - `preventSilentlyDiscardingAttributes` — the lesson 03 guard: a key that is not in `$fillable`
     throws instead of being dropped.
   - `preventLazyLoading` — new, and the reason for the exception above: every lazy load
     throws, but only for models that were loaded as part of a **list** with more than one model. Lazy
     loading `$project->client` on a single project page is still allowed, because it cannot cause N+1.
   - `preventAccessingMissingAttributes` — new: reading an attribute the model does not have throws a
     `MissingAttributeException` instead of returning `null`. That catches typos
     (`$project->budget_cents`) and columns left out of a partial select such as
     `->get(['id', 'name'])`. It only checks models loaded from the database (a model you just built
     with `new` or `make()` is not checked), and a key listed in `casts()` still returns `null`.
2. `! $this->app->isProduction()` means: on in development and tests, off in production. If a case slips
   through, production is only slower (or shows an empty value), not broken.
3. From now on, every list page you write must say in advance what it needs. That is the habit we want.

Check other list pages now: `/clients` and a client's project list. If one of them throws, it lazy loads
something in a loop — fix it with `with()` like in the next step.

---

## Step 7 — Fix it with eager loading

Change the query in `TimeEntryController::index()`:

```php
// app/Http/Controllers/TimeEntryController.php (index)
public function index(): View
{
    $entries = TimeEntry::query()
        ->with(['project.client', 'tags'])
        ->orderByDesc('started_at')
        ->limit(50)
        ->get();

    return view('time-entries.index', ['entries' => $entries]);
}
```

Reload `/time-entries` and look at the log.

**What happens:**

1. `with([...])` tells Eloquent which relations to load **together with** the entries.
2. After the main query, Eloquent collects all `project_id` values of the 50 entries and runs **one**
   query: `select * from projects where id in (1, 2, 3, 4, 5, 6)`. It then puts the right project into
   each entry.
3. `project.client` (dot = nested) does the same one level deeper: one query for all clients of those
   projects.
4. `tags` runs one query that joins the pivot table:
   `select tags.*, tag_time_entry.time_entry_id ... where tag_time_entry.time_entry_id in (...)`.
5. The view does not change at all. `$entry->project` now finds the relation already loaded and runs no
   query.

**Check it works:** the page is identical, no exception, and the log says

```
local.DEBUG: GET /time-entries: 6 queries
```

4 for the page + the session queries. **From 153 to 6.** Now the proof that it is constant: add 200 more
entries and reload.

```bash
php artisan tinker --execute="for (\$i = 0; \$i < 4; \$i++) { (new Database\Seeders\TimeEntrySeeder)->run(); }"
```

(The seeder uses `$this->command` only when there are no projects, so calling `run()` directly works
here.) The page still shows 50 entries and the log still says 6 queries. With the naive version it would
still be 153 for 50 rows — but a page with 500 rows would need 1501.

**Commit:** `perf(time-entries): eager load relations on the time sheet (153 → 6 queries)`

A commit message with the numbers is good evidence for ÕV4.

---

## Step 8 — Log a new time entry (R4)

```bash
php artisan make:request StoreTimeEntryRequest
```

```php
<?php

// app/Http/Requests/StoreTimeEntryRequest.php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
use Illuminate\Validation\Validator;

class StoreTimeEntryRequest extends FormRequest
{
    private const int MAX_SECONDS = 12 * 60 * 60;

    public function authorize(): bool
    {
        return true;
    }

    /**
     * @return array<string, array<int, mixed>>
     */
    public function rules(): array
    {
        return [
            'project_id' => [
                'required',
                'integer',
                Rule::exists('projects', 'id')->whereNull('archived_at'),
            ],
            'started_at' => ['required', 'date'],
            'ended_at' => ['bail', 'required', 'date', 'after:started_at'],
            'description' => ['required', 'string', 'max:255'],
            'billable' => ['boolean'],
        ];
    }

    /**
     * Checks that run after the rules above.
     *
     * @return array<int, callable>
     */
    public function after(): array
    {
        return [
            function (Validator $validator): void {
                if ($validator->errors()->hasAny(['started_at', 'ended_at'])) {
                    return; // the dates are missing or wrong; that error is enough
                }

                $seconds = $this->date('started_at')->diffInSeconds($this->date('ended_at'));

                if ($seconds > self::MAX_SECONDS) {
                    $validator->errors()->add('ended_at', 'A time entry can be at most 12 hours long.');
                }
            },
        ];
    }

    /**
     * @return array<string, string>
     */
    public function attributes(): array
    {
        return [
            'project_id' => 'project',
            'started_at' => 'start',
            'ended_at' => 'end',
        ];
    }

    /**
     * @return array<string, string>
     */
    public function messages(): array
    {
        return [
            'project_id.exists' => 'Choose an active project.',
            'ended_at.after' => 'The end must be after the start.',
        ];
    }
}
```

Add `create()` and `store()` to the controller:

```php
// app/Http/Controllers/TimeEntryController.php (add the imports and the two methods)
use App\Http\Requests\StoreTimeEntryRequest;
use App\Models\Project;
use Illuminate\Http\RedirectResponse;

public function create(): View
{
    return view('time-entries.create', [
        'projects' => Project::active()->with('client')->orderBy('name')->get(),
    ]);
}

public function store(StoreTimeEntryRequest $request): RedirectResponse
{
    TimeEntry::create($request->validated());

    return redirect()
        ->route('time-entries.index')
        ->with('status', 'Time entry logged.');
}
```

```blade
{{-- resources/views/time-entries/create.blade.php --}}
@extends('layouts.app')

@section('title', 'Log time')

@section('content')
    <h1>Log time</h1>

    <form method="POST" action="{{ route('time-entries.store') }}">
        @csrf

        <label for="project_id">
            Project
            <select id="project_id" name="project_id" required @error('project_id') aria-invalid="true" @enderror>
                <option value="">Choose a project…</option>
                @foreach ($projects as $project)
                    <option value="{{ $project->id }}" @selected((int) old('project_id') === $project->id)>
                        {{ $project->client->name }} — {{ $project->name }}
                    </option>
                @endforeach
            </select>
            @error('project_id') <small>{{ $message }}</small> @enderror
        </label>

        <div class="grid">
            <label for="started_at">
                Start
                <input id="started_at" name="started_at" type="datetime-local" required
                       value="{{ old('started_at') }}" @error('started_at') aria-invalid="true" @enderror>
                @error('started_at') <small>{{ $message }}</small> @enderror
            </label>

            <label for="ended_at">
                End
                <input id="ended_at" name="ended_at" type="datetime-local" required
                       value="{{ old('ended_at') }}" @error('ended_at') aria-invalid="true" @enderror>
                @error('ended_at') <small>{{ $message }}</small> @enderror
            </label>
        </div>

        <label for="description">
            Description
            <input id="description" name="description" type="text" maxlength="255" required
                   value="{{ old('description') }}" @error('description') aria-invalid="true" @enderror>
            @error('description') <small>{{ $message }}</small> @enderror
        </label>

        <label>
            <input type="hidden" name="billable" value="0">
            <input type="checkbox" name="billable" value="1" @checked(old('billable', true))>
            Billable
        </label>

        <button type="submit">Save entry</button>
    </form>

    <p><a href="{{ route('time-entries.index') }}">Back to the time sheet</a></p>
@endsection
```

**What happens:**

1. `Rule::exists('projects', 'id')->whereNull('archived_at')` checks that the project exists **and** is
   active. The `<select>` shows only active projects, but a request can contain any id.
2. `after:started_at` compares the two fields as dates. `bail` stops the rules for `ended_at` at the
   first failure, so you see one message, not three.
3. `after()` returns extra checks that Laravel runs **after** the rules. The 12-hour check needs both
   dates, so it first looks at the errors: if a date is missing or invalid, it stops. Otherwise
   `$this->date(...)` parses the input (a `datetime-local` value like `2026-09-30T09:00`) into Carbon,
   and `diffInSeconds()` gives the length. Longer than 12 hours → an error on `ended_at`. Exactly 12
   hours is allowed.
4. `$this` inside the function is the Form Request, so `$this->date(...)` reads the request input.
5. The hidden `billable=0` field comes **before** the checkbox. An unchecked checkbox sends nothing, so
   the hidden field provides `0`. A checked box sends `1` after it, and the later value wins.
6. `TimeEntry::create($request->validated())` saves exactly the validated fields. Their names match the
   columns, and the model's casts convert the text: `"2026-09-30T09:00"` becomes a date in the
   application timezone, `"0"`/`"1"` becomes `false`/`true`. All five fields are in `$fillable`.
7. `create()` loads projects **with** their client, because the option text uses `$project->client->name`.
   Without `with('client')`, `preventLazyLoading` would throw here — try it.

**Check it works:**

1. Click "Log time". Choose a project, start today 09:00, end 08:00 → "The end must be after the start."
2. Start 08:00, end 21:00 (13 h) → "A time entry can be at most 12 hours long."
3. Start 09:00, end 10:30, a description, billable unchecked → you are back on the time sheet with "Time
   entry logged." The new entry is on top (if it is the newest), with `1:30` and "(not billable)".

**Commit:** `feat(time-entries): log time entries with R4 validation`

---

## Step 9 — Format and final check

```bash
./vendor/bin/pint
php artisan route:list --path=time-entries
```

Open every list page once (`/clients`, a client's projects, `/time-entries`) and check the log: every page
has a small number of queries and no page throws `LazyLoadingViolationException`.

**Commit:** `style: apply pint` (only if Pint changed something).

---

## Troubleshooting

| What you see | Cause | Fix |
|---|---|---|
| `SQLSTATE[42P01]: Undefined table: relation "tags" does not exist` while migrating | The pivot migration runs before `tags` (file names out of order) | Rename the pivot file so its timestamp is later, then `php artisan migrate` |
| `SQLSTATE[23503]: Foreign key violation ... "time_entries_project_id_foreign"` when deleting a project | Restrict FK: the project has time entries | Archive the project (lesson 05 intermediate) or check for entries before deleting, like clients |
| `Attempted to lazy load [client] on model [App\Models\Project] but lazy loading is disabled.` | A list page touches a relation that was not eager loaded | Add it to `with()`, e.g. `with('client')` |
| `The attribute [budget_cents] either does not exist or was not retrieved for model [App\Models\Project].` | `preventAccessingMissingAttributes` (Step 6): a typo in the attribute name, or a column left out of `->get([...])` / `select(...)` | Fix the name, or add the column to the select |
| `Call to undefined method App\Models\Project::active()` | Scope missing or named wrong | The method must be `scopeActive(Builder $query)` |
| Durations like `89` instead of `90`, or negative | `diffInMinutes` in Carbon 3 returns a signed float | Call `$start->diffInMinutes($end)` (start first) and cast to `int` |
| Unchecked "Billable" is still saved as billable | Hidden `billable=0` field missing, or placed after the checkbox | Hidden field first, checkbox second |
| The log shows no "queries" lines | `APP_ENV` is not `local`, or `LOG_LEVEL` is higher than `debug` | Check `.env`: `APP_ENV=local`, `LOG_LEVEL=debug` |
| `Integrity constraint violation: duplicate key value violates unique constraint "tag_time_entry_pkey"` | `attach()` with a tag that is already attached | Use `sync()` (independent work) or `syncWithoutDetaching()` |
| `Class "Database\Seeders\TimeEntrySeeder" not found` | Composer autoload is out of date | `composer dump-autoload` |

---

## Recap

- `database/migrations/*_create_time_entries_table.php`, `*_create_tags_table.php`,
  `*_create_tag_time_entry_table.php` — FKs with restrict/cascade, composite index, pivot index.
- `app/Models/TimeEntry.php`, `app/Models/Tag.php` — `belongsTo`, `belongsToMany`, casts,
  `durationMinutes()`.
- `app/Models/Project.php` — `timeEntries()` (`hasMany`), `scopeActive()`.
- `database/seeders/TimeEntrySeeder.php` — 60 entries over three weeks, with tags.
- `app/Providers/AppServiceProvider.php` — `Model::shouldBeStrict()` (three guards, including
  `preventLazyLoading`) instead of the lesson 03 line, query count per request in the log.
- `app/Http/Controllers/TimeEntryController.php` — `index()` with `with(['project.client', 'tags'])`,
  `create()`, `store()`.
- `app/Http/Requests/StoreTimeEntryRequest.php` — R4 with `after:started_at` and an `after()` check.
- `resources/views/time-entries/index.blade.php`, `create.blade.php`; `hours_minutes()` helper.

---

## Independent work — solutions

<details>
<summary><strong>Basic — tag time entries</strong></summary>

**Rules** — add to `StoreTimeEntryRequest::rules()`:

```php
// app/Http/Requests/StoreTimeEntryRequest.php (rules, add two entries)
'tags' => ['array'],
'tags.*' => ['integer', 'distinct', Rule::exists('tags', 'id')],
```

`tags.*` applies the rules to every element of the array. `distinct` rejects the same id twice.

**Controller** — pass the tags to the form and sync them after creating the entry:

```php
// app/Http/Controllers/TimeEntryController.php (create and store; import App\Models\Tag)
public function create(): View
{
    return view('time-entries.create', [
        'projects' => Project::active()->with('client')->orderBy('name')->get(),
        'tags' => Tag::query()->orderBy('name')->get(),
    ]);
}

public function store(StoreTimeEntryRequest $request): RedirectResponse
{
    $entry = TimeEntry::create($request->safe()->except('tags'));

    $entry->tags()->sync($request->validated('tags', []));

    return redirect()
        ->route('time-entries.index')
        ->with('status', 'Time entry logged.');
}
```

**View** — add above the Billable checkbox in `create.blade.php`:

```blade
{{-- resources/views/time-entries/create.blade.php (above the billable checkbox) --}}
<label for="tags">
    Tags <small>(Ctrl/Cmd + click to choose several)</small>
    <select id="tags" name="tags[]" multiple size="5" @error('tags.*') aria-invalid="true" @enderror>
        @foreach ($tags as $tag)
            <option value="{{ $tag->id }}" @selected(in_array($tag->id, old('tags', [])))>{{ $tag->name }}</option>
        @endforeach
    </select>
    @error('tags.*') <small>{{ $message }}</small> @enderror
</label>
```

- `name="tags[]"` makes PHP collect all selected options into an array.
- `$request->safe()->except('tags')` is the validated data without `tags`: `tags` is not a column, and
  passing it to `create()` would throw (the lesson 03 guard against silently dropped attributes).
- `sync([1, 3])` makes the pivot table contain **exactly** these tags for the entry: it inserts missing
  links and deletes links that are not in the list. `attach()` only adds, `detach()` only removes. For a
  form that shows the full selection, `sync()` is the right choice — it will also work for an edit form.
- The time sheet already eager loads `tags`, so the query count does not change.

</details>

<details>
<summary><strong>Basic — weekly summary report</strong></summary>

```php
// routes/web.php
use App\Http\Controllers\ReportController;

Route::get('/reports/weekly', [ReportController::class, 'weekly'])->name('reports.weekly');
```

```php
<?php

// app/Http/Controllers/ReportController.php

namespace App\Http\Controllers;

use Carbon\CarbonImmutable;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use Illuminate\View\View;

class ReportController extends Controller
{
    public function weekly(Request $request): View
    {
        $week = $request->query('week', now()->format('o-\WW'));

        // is_string: ?week[]=x sends an array, which must be a 400, not a 500.
        if (! is_string($week) || preg_match('/^(\d{4})-W(\d{2})$/', $week, $matches) !== 1) {
            abort(400, 'Invalid week. Use the format 2026-W40.');
        }

        $from = CarbonImmutable::now()->setISODate((int) $matches[1], (int) $matches[2])->startOfDay();

        // setISODate() rolls "2026-W60" over into a later year; reject that.
        if ($from->format('o-\WW') !== $week) {
            abort(400, 'This week does not exist.');
        }

        $to = $from->addWeek();

        $rows = DB::table('time_entries')
            ->join('projects', 'projects.id', '=', 'time_entries.project_id')
            ->join('clients', 'clients.id', '=', 'projects.client_id')
            ->where('time_entries.started_at', '>=', $from)
            ->where('time_entries.started_at', '<', $to)
            ->groupBy('projects.id', 'projects.name', 'clients.name')
            ->orderBy('clients.name')
            ->orderBy('projects.name')
            ->select('projects.id', 'projects.name as project_name', 'clients.name as client_name')
            ->selectRaw('CAST(SUM(EXTRACT(EPOCH FROM (time_entries.ended_at - time_entries.started_at))) / 60 AS integer) AS minutes')
            ->get();

        return view('reports.weekly', [
            'week' => $week,
            'from' => $from,
            'to' => $to->subDay(),
            'rows' => $rows,
            'totalMinutes' => (int) $rows->sum('minutes'),
            'previousWeek' => $from->subWeek()->format('o-\WW'),
            'nextWeek' => $from->addWeek()->format('o-\WW'),
        ]);
    }
}
```

```blade
{{-- resources/views/reports/weekly.blade.php --}}
@extends('layouts.app')

@section('title', 'Weekly report '.$week)

@section('content')
    <h1>Week {{ $week }}</h1>
    <p>{{ $from->format('d.m.Y') }} – {{ $to->format('d.m.Y') }}</p>

    <nav>
        <ul>
            <li><a href="{{ route('reports.weekly', ['week' => $previousWeek]) }}">← {{ $previousWeek }}</a></li>
        </ul>
        <ul>
            <li><a href="{{ route('reports.weekly', ['week' => $nextWeek]) }}">{{ $nextWeek }} →</a></li>
        </ul>
    </nav>

    <table>
        <thead>
            <tr><th>Client</th><th>Project</th><th>Time</th></tr>
        </thead>
        <tbody>
            @forelse ($rows as $row)
                <tr>
                    <td>{{ $row->client_name }}</td>
                    <td><a href="{{ route('projects.show', $row->id) }}">{{ $row->project_name }}</a></td>
                    <td>{{ hours_minutes((int) $row->minutes) }}</td>
                </tr>
            @empty
                <tr><td colspan="3">No time logged in this week.</td></tr>
            @endforelse
        </tbody>
        <tfoot>
            <tr><th colspan="2">Total</th><th>{{ hours_minutes($totalMinutes) }}</th></tr>
        </tfoot>
    </table>
@endsection
```

Add `<li><a href="{{ route('reports.weekly') }}">Weekly report</a></li>` to the navigation.

- `now()->format('o-\WW')`: `o` is the **ISO year** and `W` the ISO week number with a leading zero.
  (`Y` would be wrong around New Year: 29 December 2025 belongs to ISO week `2026-W01`.) `\W` is a literal
  `W`.
- `setISODate(2026, 40)` jumps to Monday of ISO week 40.
- One query with `GROUP BY` returns one row per project, already summed by PostgreSQL. `EXTRACT(EPOCH
  FROM interval)` is the length in seconds; `/ 60` gives minutes. This SQL is PostgreSQL syntax.
- `$rows->sum('minutes')` adds up **the few result rows** in PHP — fine, the heavy work (reading all
  entries) was done in SQL. You could also compute the total with `GROUP BY ROLLUP`, but it is not needed.
- `->where('started_at', '>=', $from)->where('started_at', '<', $to)` is the half-open range.

</details>

<details>
<summary><strong>Intermediate — filter the time sheet</strong></summary>

```php
// app/Http/Controllers/TimeEntryController.php (replace index; add the imports)
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Http\Request;
use Illuminate\Support\Carbon;
use Illuminate\Validation\Rule;

public function index(Request $request): View
{
    $filters = $request->validate([
        'project' => ['nullable', 'integer', 'exists:projects,id'],
        'from' => ['nullable', 'date_format:Y-m-d'],
        'to' => ['nullable', 'date_format:Y-m-d', Rule::when($request->filled('from'), ['after_or_equal:from'])],
    ]);

    $entries = TimeEntry::query()
        ->with(['project.client', 'tags'])
        ->when($filters['project'] ?? null, fn (Builder $query, $projectId) => $query->where('project_id', $projectId))
        ->when($filters['from'] ?? null, fn (Builder $query, string $from) => $query->where('started_at', '>=', Carbon::parse($from)->startOfDay()))
        ->when($filters['to'] ?? null, fn (Builder $query, string $to) => $query->where('started_at', '<', Carbon::parse($to)->addDay()->startOfDay()))
        ->orderByDesc('started_at')
        ->limit(50)
        ->get();

    return view('time-entries.index', [
        'entries' => $entries,
        'projects' => Project::query()->with('client')->orderBy('name')->get(),
        'filters' => $filters,
    ]);
}
```

Filter form in `index.blade.php`, above the table:

```blade
{{-- resources/views/time-entries/index.blade.php (above the table) --}}
<form method="GET" action="{{ route('time-entries.index') }}">
    <div class="grid">
        <label for="project">
            Project
            <select id="project" name="project">
                <option value="">All projects</option>
                @foreach ($projects as $project)
                    <option value="{{ $project->id }}" @selected((int) ($filters['project'] ?? 0) === $project->id)>
                        {{ $project->client->name }} — {{ $project->name }}
                    </option>
                @endforeach
            </select>
        </label>
        <label for="from">
            From
            <input id="from" name="from" type="date" value="{{ $filters['from'] ?? '' }}">
        </label>
        <label for="to">
            To
            <input id="to" name="to" type="date" value="{{ $filters['to'] ?? '' }}"
                   @error('to') aria-invalid="true" @enderror>
            @error('to') <small>{{ $message }}</small> @enderror
        </label>
    </div>
    <button type="submit">Filter</button>
    <a href="{{ route('time-entries.index') }}">Reset</a>
</form>
```

- A **GET** form: filtering changes nothing, and the filtered URL can be bookmarked or shared.
- `when($value, fn ...)` adds the condition only when the value is not empty — no `if` chains.
- `to` is inclusive for the user, so we query `< to + 1 day 00:00`.
- `Rule::when(...)` adds `after_or_equal:from` only when `from` is present; otherwise Laravel would try
  to compare with the text "from" and fail.
- If validation fails on this GET request, Laravel redirects back with the errors, like a POST form.
- The query count is still constant: 4 for the entries + 2 for the filter's project list (projects, clients).

</details>

<details>
<summary><strong>Advanced — prove the query count is constant (feature test)</strong></summary>

Tests are the topic of lessons 11 and 14; this is a first taste. **Important:** open `phpunit.xml` and
check that these two lines are present and **not** commented out:

```xml
<env name="DB_CONNECTION" value="sqlite"/>
<env name="DB_DATABASE" value=":memory:"/>
```

`RefreshDatabase` empties the test database before each test. Without these lines the test uses your
development PostgreSQL database and **deletes your data**. With them it uses a fresh SQLite database in
memory — fast, but not identical to PostgreSQL (lesson 14 discusses this trade-off). The time sheet query
works on both.

```php
<?php

// tests/Feature/TimeSheetQueryCountTest.php

namespace Tests\Feature;

use App\Models\Project;
use App\Models\Tag;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class TimeSheetQueryCountTest extends TestCase
{
    use RefreshDatabase;

    public function test_time_sheet_with_5_entries_needs_4_queries(): void
    {
        $this->createEntries(5);

        $this->expectsDatabaseQueryCount(4);
        $this->get('/time-entries')->assertOk();
    }

    public function test_time_sheet_with_200_entries_still_needs_4_queries(): void
    {
        $this->createEntries(200);

        $this->expectsDatabaseQueryCount(4);
        $this->get('/time-entries')->assertOk();
    }

    private function createEntries(int $count): void
    {
        $project = Project::factory()->hourly(8000)->create(); // also creates a client (lesson 03 factory)
        $tag = Tag::firstOrCreate(['name' => 'meeting']);

        for ($i = 0; $i < $count; $i++) {
            $start = now()->subDays(30)->addHours($i);
            $entry = $project->timeEntries()->create([
                'started_at' => $start,
                'ended_at' => $start->copy()->addMinutes(45),
                'description' => "Entry {$i}",
            ]);
            $entry->tags()->attach($tag);
        }
    }
}
```

```bash
php artisan test --filter=TimeSheetQueryCountTest
```

- `expectsDatabaseQueryCount(4)` is built into Laravel's `TestCase`. It counts every query **from this
  line on** and checks the number when the test ends. So we call it after creating the data: the inserts
  are not counted, only the real HTTP request (`$this->get()`), including the view.
- The two tests are the proof: the same 4 queries for 5 entries and for 200 entries.
- In tests `APP_ENV=testing`, so `preventLazyLoading` is **on**: if someone removes `with()`, the page
  throws, `assertOk()` fails, and the test tells you why.
- In tests the session driver is `array` (see `phpunit.xml`), so there are no session queries: the page
  needs exactly 4.
- Try it: remove `'tags'` from `with()` and run the test again. It fails with the lazy loading message.
  Put it back.

</details>
