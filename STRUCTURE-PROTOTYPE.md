# Structure prototype — lesson 07 Laravel, Step 4

Same final code as the current step. The difference is the order: the class grows in small pieces, and
each piece starts with the problem it solves. The long "What happens" list is gone. Its points now sit
next to the lines they explain, and only a short recap is left at the end.

---

## Step 4 — Create `TimeEntryService`

Right now `TimeEntryController` does two jobs. It speaks HTTP: it reads the request and picks a redirect.
It also knows how time entries work: which queries the time sheet needs, and what must be true before
time is logged. In this step we move the second job into a class of its own. The controller keeps only
the HTTP part (step 5).

We build the service in four small pieces. After each piece the file is valid PHP, so you can stop and
run Pint at any point.

### 4a — An empty service

Create the class with Artisan:

```bash
php artisan make:class Services/TimeEntryService
```

Laravel creates `app/Services/` if it does not exist. Replace the generated content with a skeleton:

```php
<?php

// app/Services/TimeEntryService.php

namespace App\Services;

/**
 * Use cases for time entries: reading the time sheet and logging new time.
 */
class TimeEntryService
{
}
```

There is no `final` on the class, and that is on purpose. In lesson 12 we replace this service with a
test double (`$this->mock(TimeEntryService::class)`). Mockery does that by creating a subclass, and
PHP does not allow a subclass of a `final` class.

### 4b — Move the read queries

Open `TimeEntryController` next to the new file. Its `index()` and `create()` methods each build a query.
Those queries are knowledge about time entries, not about HTTP, so they move here. Add them inside the
class:

```php
    /**
     * The time sheet: newest entries first, with everything the page shows loaded up front (no N+1).
     *
     * @return Collection<int, TimeEntry>
     */
    public function timeSheet(int $limit = 50): Collection
    {
        return TimeEntry::query()
            ->with(['project.client', 'tags'])
            ->latest('started_at')
            ->limit($limit)
            ->get();
    }

    /**
     * @return Collection<int, Tag>
     */
    public function availableTags(): Collection
    {
        return Tag::query()->orderBy('name')->get();
    }
```

Add the imports at the top: `App\Models\Tag`, `App\Models\TimeEntry` and
`Illuminate\Database\Eloquent\Collection`.

Both queries are copied over unchanged; the `with(...)` you added in lesson 06 comes along. The project
dropdown needs a small change, though. The form used to list every project, including archived ones, so
users could pick a project they are not allowed to log time on. Now that the query has its own named
method, the rule fits in its name:

```php
    /**
     * Projects that can receive new time — archived ones are left out.
     *
     * @return Collection<int, Project>
     */
    public function loggableProjects(): Collection
    {
        return Project::query()
            ->with('client')
            ->whereNull('archived_at')
            ->orderBy('name')
            ->get();
    }
```

(Import `App\Models\Project`.)

### 4c — The use case: log time

Writing is where the business rule lives: **no new time on an archived project**. We start with the
check alone, so the order is clear. The method loads the project, checks it, and only then writes:

```php
    /**
     * Log a new time entry.
     *
     * @param  array<string, mixed>  $data  validated input: project_id, started_at, ended_at,
     *                                      description, billable, tags (list of tag ids, optional)
     *
     * @throws ProjectArchivedException when the project no longer accepts time
     */
    public function log(array $data): TimeEntry
    {
        $project = Project::query()->findOrFail($data['project_id']);

        if ($project->isArchived()) {
            throw ProjectArchivedException::for($project);
        }

        return TimeEntry::create(Arr::except($data, ['tags']));
    }
```

(Import `App\Exceptions\ProjectArchivedException` and `Illuminate\Support\Arr`.)

Look at what `log()` receives. It gets a plain array, not the `Request`. The controller will pass in
`$request->validated()`, but a console command or a test can just as well pass a hand-made array. The
service has no idea that HTTP exists. That is the point of this lesson.

`Arr::except(..., ['tags'])` is there because `tags` is not a column. If it reached `create()`, the
`preventSilentlyDiscardingAttributes` guard from lesson 03 would throw.

### 4d — Tags, and why we need a transaction

The entry still needs its tags. That is a second write: rows in the pivot table `tag_time_entry`. Two
writes mean a new risk. If `create()` succeeds and `sync()` fails (a tag id that does not exist, say),
the entry ends up saved without its tags. Wrap both writes in a transaction so they succeed or fail
together:

```php
    public function log(array $data): TimeEntry
    {
        return DB::transaction(function () use ($data): TimeEntry {
            $project = Project::query()->findOrFail($data['project_id']);

            if ($project->isArchived()) {
                throw ProjectArchivedException::for($project);
            }

            $entry = TimeEntry::create(Arr::except($data, ['tags']));
            $entry->tags()->sync($data['tags'] ?? []);

            return $entry;
        });
    }
```

(Import `Illuminate\Support\Facades\DB`.)

`DB::transaction()` runs the closure and commits when the closure returns. If anything inside throws,
the exception from our rule included, Laravel rolls back and re-throws the same exception. Whatever the
closure returns, `DB::transaction()` returns too, so `log()` still returns the entry.

> **Why not put the archived check in the form request?** `Rule::exists(...)->whereNull('archived_at')`
> would work for the web form. But the rule describes the *state of the business*, and it must also
> hold for a CSV import, an API endpoint or a console command. Those call the service, not the form
> request. Keep input checks in the request and business rules in the service.

**Check it works:**

```bash
./vendor/bin/pint app/Services
php artisan tinker --execute="dump(app(App\Services\TimeEntryService::class)->loggableProjects()->pluck('name'));"
```

The list does not contain archived projects. Nothing calls `log()` yet; step 5 wires it into the
controller.

**Recap:** the service owns the queries and the "log time" use case. It takes plain data, enforces the
archived rule before writing anything, and writes the entry and its tags in one transaction.

---

## What changed, as a pattern for every step

| Now | Proposed |
|---|---|
| **Why:** paragraph → one big code block → numbered "What happens" list | Problem in 2–3 sentences → small code piece → 1–3 sentences on what to notice → next piece |
| The learner types 80 lines before learning why any of them exist | Each piece is ≤ ~30 lines and starts with the reason it exists |
| Explanations point back at code by number ("3. `DB::transaction`…") | The explanation sits right under the lines it is about |
| A side remark ("why not final") sits after the "What happens" list | Side remarks sit where the decision is made, or in a `>` aside |
| Final full listing only | The final full file can still be shown once, in a collapsed `<details>`, for students who fall behind |

Rules of thumb for the rewrite:
1. **Open each step with the problem, not the solution.** Name what is wrong or missing in the app right now.
2. **Grow code in pieces where the reason changes.** Split where a new "why" starts, not at a fixed length. A 150-line test class becomes "first test → what it proves → the next case".
3. **Bridge every pair of code blocks.** At least one sentence between them: what to notice, or why the next piece is needed. Never two fences back to back.
4. **Keep "What happens" short or drop it.** If the points were moved next to the code, end with a 1–2 line recap.
5. **Keep the house rhythm**: Check it works → Commit. That part already flows well.
6. **The concept README keeps the theory; the walkthrough keeps the doing.** Repeat a concept in the walkthrough only as a one-line pointer.
