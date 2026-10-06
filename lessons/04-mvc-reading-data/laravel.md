# 04 — MVC: controllers and views · Laravel

[← Concepts](README.md) · Starting point: end of lesson 03 · Estimated time in class: 2 h

## What we build today

- resource routes for clients (`index`, `show`)
- `ClientController` with `index()` (paginated, with project counts) and `show()` (client + projects)
- a global helper `euros(?int $cents)` that formats cents as `1 500,00 €` with Laravel's `Number::currency()`
- a "Clients" link in the navigation; the layout's flash block moved into a partial
- views `clients/index`, `clients/show`, a pagination partial (the default for every `links()` call), and
  a custom 404 page

**Final result:** `/clients` shows a paginated table of clients; clicking a name opens `/clients/{id}`
with the client's details and projects; `/clients/999` shows a styled "Not found" page with status 404.

Start the database (`docker compose up -d`) and reset the data so it matches lesson 03:

```bash
php artisan migrate:fresh --seed
```

---

## Step 1 — Routes

**Why:** before writing a controller, decide which URLs exist. We use Laravel's **resource routes**, which
follow the RESTful naming convention from [Concepts §2](README.md#2-routing-and-restful-resource-names).
Today we need only two of the seven actions.

Open `routes/web.php` and add, below the existing home and about routes:

```php
<?php

// routes/web.php

use App\Http\Controllers\ClientController;
use App\Http\Controllers\HomeController;
use Illuminate\Support\Facades\Route;

Route::get('/', [HomeController::class, 'index'])->name('home');
// ... your about route from lesson 01 stays here

Route::resource('clients', ClientController::class)->only(['index', 'show']);
```

(Keep your existing lines as they are; the new parts are the `use App\Http\Controllers\ClientController;`
import and the `Route::resource(...)` line.)

**What happens:**

1. `Route::resource('clients', ...)` would register all seven RESTful routes. `->only(['index', 'show'])`
   registers just two. In lesson 05 you remove `->only(...)` (or extend the list).
2. Each route gets a **name**: `clients.index` and `clients.show`. Views use these names to build URLs.
3. The show route's parameter is called `{client}` — the singular of the resource name. That name matters
   in step 2.

**Check it works:**

```bash
php artisan route:list --path=clients
```

```
  GET|HEAD  clients ............ clients.index › ClientController@index
  GET|HEAD  clients/{client} ... clients.show › ClientController@show
```

(Laravel complains that `ClientController` does not exist if you open the URL now — that is the next step.)

---

## Step 2 — `ClientController`

```bash
php artisan make:controller ClientController
```

We do **not** use `--resource` here: it would create seven empty methods, and five of them would stay
empty until lesson 05. Empty methods that "do nothing" are confusing. Replace the file:

```php
<?php

// app/Http/Controllers/ClientController.php

namespace App\Http\Controllers;

use App\Models\Client;
use Illuminate\View\View;

class ClientController extends Controller
{
    public function index(): View
    {
        $clients = Client::query()
            ->withCount('projects')
            ->orderBy('name')
            ->orderBy('id')
            ->paginate(10);

        return view('clients.index', ['clients' => $clients]);
    }

    public function show(Client $client): View
    {
        $projects = $client->projects()->orderBy('name')->get();

        return view('clients.show', [
            'client' => $client,
            'projects' => $projects,
        ]);
    }
}
```

**What happens in `index()`:**

1. `Client::query()` starts a query builder. (It is the same as starting with `Client::withCount(...)`,
   but reads better when the chain is long.)
2. `->withCount('projects')` adds a **sub-query** to the SQL that counts the projects of each client. Every
   client model gets an extra attribute `projects_count`. The counting happens in the database, in the
   same query — no extra query per client (see [Concepts §8](README.md#8-a-preview-counting-projects-without-n1)).
3. `->orderBy('name')->orderBy('id')` — a paginated list must be sorted. The `id` is a tie-breaker: if two
   clients have the same name, their order is still fixed.
4. `->paginate(10)` reads `?page=` from the current request (default 1), runs `LIMIT 10 OFFSET ...`, and a
   second query `select count(*)` for the total. It returns a `LengthAwarePaginator`: a list of up to 10
   clients plus page information.
5. `view('clients.index', [...])` renders `resources/views/clients/index.blade.php` with a variable
   `$clients`.

**What happens in `show()` — route model binding:**

1. The route is `clients/{client}` and the parameter is typed `Client $client`. The names match, so Laravel
   runs `Client::where('id', $value)->firstOrFail()` **before** calling the method.
2. If there is no such client, Laravel throws `ModelNotFoundException`, turns it into a **404** response,
   and `show()` never runs. You write no `if (! $client)` code.
3. `$client->projects()` — with parentheses — is a query builder, so we can add `orderBy('name')` before
   `get()`. The controller loads the projects and passes them to the view; the view does not query.

The SQL for `/clients` (you can see it later with the query log):

```sql
select count(*) as aggregate from "clients"
select "clients".*, (select count(*) from "projects" where "clients"."id" = "projects"."client_id") as "projects_count"
  from "clients" order by "name" asc, "id" asc limit 10 offset 0
```

---

## Step 3 — The `euros()` helper

**Why:** the client page shows rates and prices. They are stored in cents. A small **global function**
turns cents into text like `1 500,00 €` and is easy to call from Blade. We do not write the formatting rules
ourselves: Laravel's `Number::currency()` already knows how Estonian writes money (comma for decimals,
space between thousands, `€` after the number). Lesson 10 replaces this helper with a `Money` class.

Create the file:

```php
<?php

// app/helpers.php

use Illuminate\Support\Number;

if (! function_exists('euros')) {
    /**
     * Format integer cents as euros, e.g. 150000 → "1 500,00 €". Null → "—".
     * Temporary: replaced by App\Support\Money in lesson 10.
     */
    function euros(?int $cents): string
    {
        if ($cents === null) {
            return '—';
        }

        return Number::currency($cents / 100, in: 'EUR', locale: 'et');
    }
}
```

Tell Composer to load it on every request. Open `composer.json` and add a `files` entry inside the
existing `autoload` section:

```json
// composer.json — inside "autoload" (keep the existing "psr-4" block)
"autoload": {
    "psr-4": {
        "App\\": "app/",
        "Database\\Factories\\": "database/factories/",
        "Database\\Seeders\\": "database/seeders/"
    },
    "files": [
        "app/helpers.php"
    ]
},
```

(JSON has no comments — the first line above is only a label for you; do not copy it into the file.)

```bash
composer dump-autoload
```

**What happens:**

1. `$cents / 100` turns `1205` into the float `12.05`. That is fine here, because it is only **for
   display**: rule R1 (no floats) is about storing and calculating money. Nothing is stored or added up,
   the number is only printed. A float is exact enough for any realistic amount, and the formatter rounds
   to 2 decimals.
2. `Number::currency(..., in: 'EUR', locale: 'et')` uses PHP's `intl` extension (the ICU library). For the
   Estonian locale it prints `1 500,00 €`: a comma for decimals, a space between thousands, `€` at the end.
   A negative amount gets a minus sign: `−5,00 €`.
3. The "spaces" in the output are **non-breaking spaces** (U+00A0), not normal spaces. The browser shows
   them the same, but it never breaks the line between `1 500,00` and `€`. In a test, compare with
   `"1\u{00A0}500,00\u{00A0}€"`, not with a string you typed with the space bar.
4. `null` becomes `—`: a fixed-price project has no hourly rate, and the page should show a dash, not
   `0,00 €` (which would be a lie).
5. `if (! function_exists('euros'))` prevents a fatal error if the file is loaded twice.
6. The `files` autoload entry makes Composer `require` the file once at startup, so `euros()` is available
   everywhere, including Blade.

`Number::currency()` needs the PHP `intl` extension. Check that you have it:

```bash
php -m | grep intl
```

No output means it is missing — see the troubleshooting table.

**Check it works:**

```bash
php artisan tinker --execute="echo euros(150000), ' | ', euros(1205), ' | ', euros(null);"
```

Output: `1 500,00 € | 12,05 € | —` (the spaces inside the amounts are non-breaking spaces).

---

## Step 4 — Navigation link, and the flash block as a partial

Open `resources/views/layouts/app.blade.php` (from lesson 01). Find the second `<ul>` inside `<nav>` and
add a Clients link between Home and About:

```blade
{{-- resources/views/layouts/app.blade.php — the second <ul> inside <nav> --}}
<ul>
    <li>
        <a href="{{ route('home') }}" @if (request()->routeIs('home')) aria-current="page" @endif>Home</a>
    </li>
    <li>
        <a href="{{ route('clients.index') }}" @if (request()->routeIs('clients.*')) aria-current="page" @endif>Clients</a>
    </li>
    <li>
        <a href="{{ route('about') }}" @if (request()->routeIs('about')) aria-current="page" @endif>About</a>
    </li>
</ul>
```

Since lesson 01 the layout also has a flash block inside `<main>`:

```blade
@if (session('status'))
    <article role="status">{{ session('status') }}</article>
@endif
```

We move it into a **partial** of its own. The layout gets shorter, and in lesson 05 you can style or extend
the flash area (for example with error messages) in one small file. Create the partial:

```blade
{{-- resources/views/partials/flash.blade.php --}}
@if (session('status'))
    <article role="status">{{ session('status') }}</article>
@endif
```

and replace the block in the layout with an include, so `<main>` looks like this:

```blade
{{-- resources/views/layouts/app.blade.php — inside <body> --}}
<main class="container">
    @include('partials.flash')

    @yield('content')
</main>
```

**What happens:**

1. `route('clients.index')` builds the URL `/clients` from the route **name**. If the URL changes one day,
   this link follows.
2. `request()->routeIs('clients.*')` is true on every client page (`clients.index`, `clients.show`, and in
   lesson 05 `clients.create`, `clients.edit`). `aria-current="page"` tells screen readers — and Pico's
   styling — which menu item is active.
3. `@include('partials.flash')` inserts `resources/views/partials/flash.blade.php`. The dot in the name is a
   folder separator. The included view sees the same variables as the page.
4. `session('status')` reads a **flash message**: a value stored in the session for exactly one following
   request. Nothing sets it yet; lesson 05 does, with `redirect()->route(...)->with('status', 'Client
   saved.')`. Until then, the partial prints nothing.

**Check it works:** reload the home page. It looks exactly as before, and the menu has a "Clients" link.

---

## Step 5 — The client list view and a pagination partial

```blade
{{-- resources/views/clients/index.blade.php --}}
@extends('layouts.app')

@section('title', 'Clients')

@section('content')
    <h1>Clients</h1>

    @if ($clients->isEmpty())
        <p>No clients yet.</p>
    @else
        <p>Showing {{ $clients->firstItem() }}–{{ $clients->lastItem() }} of {{ $clients->total() }} clients.</p>

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
                @foreach ($clients as $client)
                    <tr>
                        <td><a href="{{ route('clients.show', $client) }}">{{ $client->name }}</a></td>
                        <td>{{ $client->email }}</td>
                        <td>{{ $client->vat_number ?? '—' }}</td>
                        <td>{{ $client->projects_count }}</td>
                    </tr>
                @endforeach
            </tbody>
        </table>

        {{ $clients->links() }}
    @endif
@endsection
```

Laravel's default pagination links are styled for Tailwind CSS, which we do not use. We write our own
small partial that fits Pico.css:

```blade
{{-- resources/views/partials/pagination.blade.php --}}
@if ($paginator->hasPages())
    <nav aria-label="Pagination">
        <ul>
            <li>
                @if ($paginator->onFirstPage())
                    <span aria-disabled="true">← Previous</span>
                @else
                    <a href="{{ $paginator->previousPageUrl() }}" rel="prev">← Previous</a>
                @endif
            </li>
        </ul>
        <ul>
            <li>Page {{ $paginator->currentPage() }} of {{ $paginator->lastPage() }}</li>
        </ul>
        <ul>
            <li>
                @if ($paginator->hasMorePages())
                    <a href="{{ $paginator->nextPageUrl() }}" rel="next">Next →</a>
                @else
                    <span aria-disabled="true">Next →</span>
                @endif
            </li>
        </ul>
    </nav>
@endif
```

Tell Laravel to use this partial for **every** `links()` call, so no view has to name it. Open
`app/Providers/AppServiceProvider.php` and add one line to `boot()` (keep the lesson 03 line):

```php
// app/Providers/AppServiceProvider.php — add the import at the top
use Illuminate\Pagination\Paginator;

public function boot(): void
{
    Model::preventSilentlyDiscardingAttributes(! $this->app->isProduction()); // lesson 03
    Paginator::defaultView('partials.pagination');
}
```

**What happens:**

1. `@extends('layouts.app')` uses the layout; `@section('content')` … `@endsection` is the part that goes
   into `@yield('content')`. `@section('title', 'Clients')` fills the title (if your layout yields one).
2. `@if ($clients->isEmpty())` — the **empty state**. Without it, a new installation shows an empty table.
3. `firstItem()`, `lastItem()`, `total()` come from the paginator: "Showing 1–3 of 3 clients".
4. `{{ ... }}` **escapes** everything it prints. A client name containing `<script>` is shown as text
   (step 8 proves it).
5. `route('clients.show', $client)` — you pass the model; Laravel takes its id and builds `/clients/1`.
6. `$client->projects_count` is the attribute added by `withCount` — no query here.
7. `$client->vat_number ?? '—'` shows a dash for `null`. This is display logic, which is fine in a view.
8. `$clients->links()` renders our partial (thanks to `Paginator::defaultView(...)`) and passes the
   paginator in as `$paginator`. `previousPageUrl()` and `nextPageUrl()` build `?page=N` links.
   `hasPages()` is false when everything fits on one page, so with 3 clients you see no pagination yet.

**Check it works:** open `/clients`. You see Acme OÜ, Birch & Daughters and Kuressaare Kala AS with 3, 2
and 2 projects, and "Showing 1–3 of 3 clients".

To see pagination working, add some extra clients in Tinker:

```bash
php artisan tinker --execute="App\Models\Client::factory()->count(25)->create();"
```

Reload `/clients`: now there are 3 pages, and "Next →" works. (Run `php artisan migrate:fresh --seed` later
to return to the standard data.)

**Commit:** `feat(clients): paginated clients list`

---

## Step 6 — The client detail view

```blade
{{-- resources/views/clients/show.blade.php --}}
@extends('layouts.app')

@section('title', $client->name)

@section('content')
    <p><a href="{{ route('clients.index') }}">← All clients</a></p>

    <h1>{{ $client->name }}</h1>

    <table>
        <tbody>
            <tr>
                <th scope="row">Email</th>
                <td><a href="mailto:{{ $client->email }}">{{ $client->email }}</a></td>
            </tr>
            <tr>
                <th scope="row">VAT number</th>
                <td>{{ $client->vat_number ?? '—' }}</td>
            </tr>
            <tr>
                <th scope="row">Client since</th>
                <td>{{ $client->created_at->format('d.m.Y') }}</td>
            </tr>
        </tbody>
    </table>

    <h2>Projects</h2>

    @if ($projects->isEmpty())
        <p>This client has no projects yet.</p>
    @else
        <table>
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
                @foreach ($projects as $project)
                    <tr>
                        <td>{{ $project->name }}</td>
                        <td>{{ $project->billing_type->label() }}</td>
                        <td>{{ euros($project->hourly_rate_cents) }}</td>
                        <td>{{ euros($project->fixed_price_cents) }}</td>
                        <td>{{ euros($project->budget_cap_cents) }}</td>
                        <td>{{ $project->rounding_minutes }} min</td>
                        <td>{{ $project->isArchived() ? 'Archived' : 'Active' }}</td>
                    </tr>
                @endforeach
            </tbody>
        </table>
    @endif
@endsection
```

**What happens:**

1. `$client` is the model from route model binding; `$projects` is the collection the controller loaded.
   The view uses only these two variables.
2. `$client->created_at->format('d.m.Y')` — `created_at` is a Carbon date object (lesson 03), so we can
   format it: `30.09.2026`, the Estonian date format.
3. `$project->billing_type->label()` — `billing_type` is cast to the `BillingType` enum, so we call its
   `label()` method: "Capped hourly".
4. `euros(...)` formats cents, and prints `—` for `null`.
5. `$project->isArchived()` asks the model. The view does not check `archived_at !== null` itself; the rule
   has one home.
6. A `<table>` with `<th scope="row">` is a simple, accessible way to show label–value pairs with Pico.

**Check it works:** click "Acme OÜ" on the list. You see its email, VAT number `EE101234567`, and three
projects. "Support retainer" shows `Capped hourly`, `60,00 €`, `—`, `2 000,00 €`, `6 min`, `Active`; "Old
intranet" shows `Archived`.

**Commit:** `feat(clients): client detail page with projects`

---

## Step 7 — A custom 404 page

Open `/clients/999`. Laravel already returns status 404 (route model binding found nothing), but the page
is Laravel's default. We want our layout.

Create:

```blade
{{-- resources/views/errors/404.blade.php --}}
@extends('layouts.app')

@section('title', 'Not found')

@section('content')
    <h1>Not found</h1>
    <p>The page or record you asked for does not exist. It may have been deleted, or the link is wrong.</p>
    <p><a href="{{ route('home') }}">Back to the home page</a></p>
@endsection
```

**What happens:**

1. When Laravel needs to send an HTTP error with status 404, it looks for a view named `errors.404`. If it
   exists, it renders that view **and keeps the status 404**.
2. This works for every kind of 404: a missing client (`ModelNotFoundException`), an unknown URL like
   `/nothing-here`, or an explicit `abort(404)` in a controller.
3. The page tells the user what happened and offers a way back — it does not show technical details.

If your home route has a different name than `home`, use that name.

Now try `/clients/abc`. You do **not** get the 404 page — you get a 500 error:

```
SQLSTATE[22P02]: Invalid text representation: 7 ERROR:  invalid input syntax for type bigint: "abc"
```

Route model binding runs `where id = 'abc'`, and PostgreSQL refuses to compare a `bigint` column with
text. (SQLite and MySQL silently convert, so tutorials written for them never show this.) Tell the router
that these parameters are numbers. Add at the top of the routes, below the `use` lines:

```php
// routes/web.php — below the use lines, above the first route
Route::pattern('client', '[0-9]+');
Route::pattern('project', '[0-9]+');
```

`Route::pattern` applies to every route registered **after** it that has a parameter with that name.
`/clients/abc` no longer matches any route, so Laravel answers 404 before it touches the database. When
later lessons add a resource with a numeric id (`{time_entry}`, `{invoice}`), add a line for it too.

**Check it works:**

- `/clients/999` shows the page inside your layout.
- In the browser's developer tools (Network tab) the status is **404**.
- `curl -I http://127.0.0.1:8000/clients/999` prints `HTTP/1.1 404 Not Found`.
- `curl -I http://127.0.0.1:8000/clients/abc` also prints `HTTP/1.1 404 Not Found`.

**Commit:** `feat(errors): custom 404 page`

---

## Step 8 — Prove that output is escaped

**Why:** XSS is one of the most common web vulnerabilities. Let us see the protection working.

```bash
php artisan tinker --execute="App\Models\Client::factory()->create(['name' => '<script>alert(\"hi\")</script>']);"
```

Open `/clients`. The name is shown as the text `<script>alert("hi")</script>` — no alert pops up. View the
page source (Ctrl+U): it contains `&lt;script&gt;alert(&quot;hi&quot;)&lt;/script&gt;`.

Now, **only as an experiment**, change `{{ $client->name }}` to `{!! $client->name !!}` in
`clients/index.blade.php` and reload. The alert appears: the browser ran the "name" as JavaScript. An
attacker would not show an alert — they would send your session somewhere. **Change it back to `{{ }}`.**

Reset the data:

```bash
php artisan migrate:fresh --seed
./vendor/bin/pint --test
```

---

## Troubleshooting

| Error / symptom | Cause | Fix |
|---|---|---|
| `Target class [ClientController] does not exist.` | Missing `use App\Http\Controllers\ClientController;` in `routes/web.php` | Add the import |
| `Route [clients.index] not defined.` | Routes not registered, or a typo in the name | Check `php artisan route:list --path=clients` |
| `Call to undefined function euros()` | Composer does not load `app/helpers.php` | Check the `files` entry in `composer.json`, run `composer dump-autoload` |
| `The "intl" PHP extension is required to use the [currency] method.` | PHP has no `intl` extension | Install it (`sudo apt install php8.4-intl`, Herd and Homebrew PHP include it), enable `extension=intl` in `php.ini`, check with `php -m \| grep intl` |
| `View [clients.index] not found.` | File in the wrong place or wrong name | It must be `resources/views/clients/index.blade.php` |
| `Undefined variable $clients` | The controller passed a different name | The array key in `view(..., ['clients' => ...])` is the variable name |
| `/clients/1` shows 404 although client 1 exists | The route parameter and the method parameter have different names (`{client}` vs `$id`) or no type hint | Keep `{client}` and `Client $client` |
| Every `show` shows an **empty** client with no name | The parameter is typed but named differently (`Client $item`) — Laravel injects a new empty model | Rename the parameter to `$client` |
| `Call to a member function label() on string` | `billing_type` is not cast to the enum | Check `casts()` in `Project` (lesson 03) |
| Pagination links are huge and unstyled | Laravel uses its default Tailwind markup | Add `Paginator::defaultView('partials.pagination')` to `AppServiceProvider::boot()` (step 5), or pass the view: `links('partials.pagination')` |
| The 404 page shows Laravel's default design | The view is not at `resources/views/errors/404.blade.php` | Check path and name |

---

## Recap

- `routes/web.php` — `Route::resource('clients', ...)->only(['index', 'show'])`, named routes, `Route::pattern` for numeric ids.
- `app/Http/Controllers/ClientController.php` — thin `index()` (paginate + `withCount`) and `show()`
  (route model binding, projects loaded in the controller).
- `app/helpers.php` + `composer.json` `files` — `euros()` with `Number::currency()` (temporary until lesson 10).
- `resources/views/layouts/app.blade.php` — Clients link; the flash block moved to a partial.
- `resources/views/partials/flash.blade.php`, `partials/pagination.blade.php` — reusable pieces;
  `Paginator::defaultView('partials.pagination')` in `AppServiceProvider` makes `links()` use ours.
- `resources/views/clients/index.blade.php`, `clients/show.blade.php` — display only, escaped output,
  empty states.
- `resources/views/errors/404.blade.php` — custom 404 with the right status.

---

## Independent work — solutions

<details>
<summary><strong>Basic</strong> — <code>ProjectController</code> with shallow nesting</summary>

Routes — add to `routes/web.php` (import `App\Http\Controllers\ProjectController`):

```php
// routes/web.php
Route::resource('clients.projects', ProjectController::class)
    ->shallow()
    ->only(['index', 'show']);
```

`php artisan route:list --path=projects` shows:

```
  GET|HEAD  clients/{client}/projects ... clients.projects.index › ProjectController@index
  GET|HEAD  projects/{project} .......... projects.show › ProjectController@show
```

`clients.projects` means "projects nested in clients". `->shallow()` keeps the client id only where it is
needed: the list needs it, a single project does not (the project knows its client).

```php
<?php

// app/Http/Controllers/ProjectController.php

namespace App\Http\Controllers;

use App\Models\Client;
use App\Models\Project;
use Illuminate\View\View;

class ProjectController extends Controller
{
    public function index(Client $client): View
    {
        $projects = $client->projects()->orderBy('name')->get();

        return view('projects.index', ['client' => $client, 'projects' => $projects]);
    }

    public function show(Project $project): View
    {
        $project->load('client');

        return view('projects.show', ['project' => $project]);
    }
}
```

`$project->load('client')` loads the client in the controller, so the view does not trigger a query when it
prints `$project->client->name`. (For one project it is one query either way — but the habit "the
controller loads what the view needs" is the point.)

```blade
{{-- resources/views/projects/index.blade.php --}}
@extends('layouts.app')

@section('title', 'Projects of '.$client->name)

@section('content')
    <p><a href="{{ route('clients.show', $client) }}">← {{ $client->name }}</a></p>

    <h1>Projects of {{ $client->name }}</h1>

    @if ($projects->isEmpty())
        <p>This client has no projects yet.</p>
    @else
        <table>
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
                @foreach ($projects as $project)
                    <tr>
                        <td><a href="{{ route('projects.show', $project) }}">{{ $project->name }}</a></td>
                        <td>{{ $project->billing_type->label() }}</td>
                        <td>{{ euros($project->hourly_rate_cents) }}</td>
                        <td>{{ euros($project->fixed_price_cents) }}</td>
                        <td>{{ euros($project->budget_cap_cents) }}</td>
                        <td>{{ $project->rounding_minutes }} min</td>
                        <td>{{ $project->isArchived() ? 'Archived' : 'Active' }}</td>
                    </tr>
                @endforeach
            </tbody>
        </table>
    @endif
@endsection
```

```blade
{{-- resources/views/projects/show.blade.php --}}
@extends('layouts.app')

@section('title', $project->name)

@section('content')
    <p><a href="{{ route('clients.projects.index', $project->client) }}">← Projects of {{ $project->client->name }}</a></p>

    <h1>{{ $project->name }}</h1>

    <table>
        <tbody>
            <tr>
                <th scope="row">Client</th>
                <td><a href="{{ route('clients.show', $project->client) }}">{{ $project->client->name }}</a></td>
            </tr>
            <tr><th scope="row">Billing</th><td>{{ $project->billing_type->label() }}</td></tr>
            <tr><th scope="row">Hourly rate</th><td>{{ euros($project->hourly_rate_cents) }}</td></tr>
            <tr><th scope="row">Fixed price</th><td>{{ euros($project->fixed_price_cents) }}</td></tr>
            <tr><th scope="row">Budget cap</th><td>{{ euros($project->budget_cap_cents) }}</td></tr>
            <tr><th scope="row">Rounding</th><td>{{ $project->rounding_minutes }} min</td></tr>
            <tr><th scope="row">Status</th><td>{{ $project->isArchived() ? 'Archived' : 'Active' }}</td></tr>
            <tr><th scope="row">Created</th><td>{{ $project->created_at->format('d.m.Y') }}</td></tr>
        </tbody>
    </table>
@endsection
```

In `clients/show.blade.php`, next to the "Projects" heading, add the link and make project names link to
the project page:

```blade
{{-- resources/views/clients/show.blade.php — replace the <h2>Projects</h2> line --}}
<h2>Projects <small><a href="{{ route('clients.projects.index', $client) }}">(open list)</a></small></h2>

{{-- …and in the table row, replace the first cell --}}
<td><a href="{{ route('projects.show', $project) }}">{{ $project->name }}</a></td>
```

The two tables now repeat each other. If that bothers you, move the table into a partial
`resources/views/projects/_table.blade.php` and `@include('projects._table', ['projects' => $projects])`
from both views — a good use of partials.

Check: `/clients/1/projects` lists 3 projects; `/projects/1` shows Website redesign with a link to Acme OÜ;
`/clients/999/projects` and `/projects/999` return your 404 page (`curl -I` shows 404).

</details>

<details>
<summary><strong>Intermediate</strong> — Search by name</summary>

Controller — change `index()` to take the request (import `Illuminate\Http\Request`):

```php
// app/Http/Controllers/ClientController.php — replace index()
    public function index(Request $request): View
    {
        $search = trim($request->string('q')->toString());

        $clients = Client::query()
            ->withCount('projects')
            ->when($search !== '', fn ($query) => $query->whereLike('name', '%'.addcslashes($search, '%_\\').'%'))
            ->orderBy('name')
            ->orderBy('id')
            ->paginate(10)
            ->withQueryString();

        return view('clients.index', ['clients' => $clients, 'search' => $search]);
    }
```

1. `$request->string('q')` reads `?q=` as a `Stringable` (empty if missing); `trim` removes spaces.
2. `->when($condition, fn)` adds the `where` only when there is a search. No `if` block needed.
3. `whereLike('name', '%kala%')` is case-insensitive by default; on PostgreSQL Laravel sends
   `"name"::text ilike ?`. The value is a **bound parameter** — no SQL injection. (Older tutorials use
   `where('name', 'ilike', ...)`, which works only on PostgreSQL.)
   `addcslashes($search, '%_\\')` puts a backslash in front of `%`, `_` and `\`. In LIKE, `%` and `_` are
   wildcards; without escaping, a search for `_` would match every client. Laravel does not escape them for
   you (Spring Data does). PostgreSQL uses `\` as the default LIKE escape character.
4. `->withQueryString()` makes the pagination links keep `q=kala`.

View — add the form above the table and change the empty state:

```blade
{{-- resources/views/clients/index.blade.php — directly under <h1>Clients</h1> --}}
<form method="get" action="{{ route('clients.index') }}" role="search">
    <input type="search" name="q" value="{{ $search }}" placeholder="Search by name" aria-label="Search clients by name">
    <button type="submit">Search</button>
</form>
```

```blade
{{-- resources/views/clients/index.blade.php — replace the empty-state paragraph --}}
@if ($clients->isEmpty())
    @if ($search !== '')
        <p>No clients match "{{ $search }}".</p>
    @else
        <p>No clients yet.</p>
    @endif
@else
```

- `value="{{ $search }}"` keeps the text in the box, **escaped** (a search for `"><script>` cannot break out
  of the attribute).
- `method="get"`: a search does not change data, so it is a GET request; the URL can be bookmarked and
  shared.
- Pico styles `<form role="search">` as one line with the button attached.

Check: `/clients?q=KALA` shows Kuressaare Kala AS; `/clients?q=<b>x</b>` shows *No clients match
"&lt;b&gt;x&lt;/b&gt;"* as plain text. With 25 extra factory clients, page 2 of a search keeps `q` in the URL.

</details>

<details>
<summary><strong>Advanced</strong> — Sortable columns with a whitelist</summary>

```php
// app/Http/Controllers/ClientController.php — add the constant and replace index()

    /**
     * Allowed values of ?sort= mapped to database columns. Column names cannot be bound
     * parameters, so anything not in this list must never reach orderBy().
     */
    private const SORTABLE = [
        'name' => 'name',
        'email' => 'email',
        'vat' => 'vat_number',
    ];

    public function index(Request $request): View
    {
        $search = trim($request->string('q')->toString());
        $sort = $request->query('sort');
        $sort = is_string($sort) && array_key_exists($sort, self::SORTABLE) ? $sort : 'name';
        $direction = $request->query('direction') === 'desc' ? 'desc' : 'asc';

        $clients = Client::query()
            ->withCount('projects')
            ->when($search !== '', fn ($query) => $query->whereLike('name', '%'.addcslashes($search, '%_\\').'%'))
            ->orderBy(self::SORTABLE[$sort], $direction)
            ->orderBy('id')
            ->paginate(10)
            ->withQueryString();

        return view('clients.index', [
            'clients' => $clients,
            'search' => $search,
            'sort' => $sort,
            'direction' => $direction,
        ]);
    }
```

A partial for one sortable header:

```blade
{{-- resources/views/partials/sort-header.blade.php --}}
@php($isActive = $sort === $column)
@php($next = $isActive && $direction === 'asc' ? 'desc' : 'asc')
<th scope="col" @if ($isActive) aria-sort="{{ $direction === 'asc' ? 'ascending' : 'descending' }}" @endif>
    <a href="{{ route('clients.index', ['q' => $search, 'sort' => $column, 'direction' => $next]) }}">
        {{ $label }}@if ($isActive) {{ $direction === 'asc' ? '▲' : '▼' }}@endif
    </a>
</th>
```

Use it in the table header:

```blade
{{-- resources/views/clients/index.blade.php — replace the first three <th> --}}
@include('partials.sort-header', ['column' => 'name', 'label' => 'Name'])
@include('partials.sort-header', ['column' => 'email', 'label' => 'Email'])
@include('partials.sort-header', ['column' => 'vat', 'label' => 'VAT number'])
```

The search form must also keep the sort. Add two hidden fields inside it:

```blade
<input type="hidden" name="sort" value="{{ $sort }}">
<input type="hidden" name="direction" value="{{ $direction }}">
```

Why a whitelist (the text for your comment or `docs/`):

> Values in a query can be sent as bound parameters (`where name = ?`), so user input never becomes SQL.
> Column names and sort directions cannot be parameters — they are part of the SQL text itself. Laravel
> quotes column names in `orderBy`, which blocks the classic injection, but an unknown column still causes
> a 500 error, reveals column names, and lets users sort by columns you never meant to expose. With
> `orderByRaw` or raw SQL it becomes a real injection. A whitelist maps a small set of public names
> (`name`, `email`, `vat`) to real columns, and everything else falls back to a safe default.

`is_string` matters: `?sort[]=x` makes `query('sort')` an array, and `array_key_exists` with an array
key would crash with a 500.

Notice the public name `vat` differs from the column `vat_number`: the URL does not need to reveal the
schema.

Check: `/clients?sort=email&direction=desc` sorts by email Z–A; `/clients?sort=id;drop` and
`/clients?sort=created_at` quietly sort by name; `/clients?direction=sideways` sorts ascending.

</details>
