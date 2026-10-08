# 01 — Frameworks and the request lifecycle · Laravel

[← Concepts](README.md) · Starting point: nothing — this lesson creates the project · Estimated time in class: 2 h · Independent work tasks: [README](README.md#independent-work-2-h)

---

## What we build today

- A new Laravel project called `billable` (Laravel 12 or newer, PHP 8.4, PHPUnit, PostgreSQL).
- A `compose.yaml` that runs PostgreSQL 17 in Docker.
- The database connection configured in `.env`, and Laravel's default migrations run.
- A layout `resources/views/layouts/app.blade.php` with Pico.css.
- `HomeController` and `resources/views/home.blade.php`: a home page that shows **Billable** and
  today's date, where the date is supplied by the controller.
- A Git repository on GitHub with the first commits.

When you are done, `http://127.0.0.1:8000` shows a clean, styled page with a navigation bar, the title
"Billable" and a sentence like "Today is 30.09.2026."

---

## Step 0 — Check your tools

**Why:** half of the problems in the first lesson are missing or old tools. Checking first takes two
minutes and saves twenty.

Open a terminal (on Windows: PowerShell or the Herd terminal) and run each command:

> **Windows with WSL?** Choose one place to work and use it for everything. If you work in WSL,
> install PHP, Composer and the Laravel installer **inside Ubuntu**, not with Herd. Herd runs on Windows
> only, and a WSL terminal cannot use it. Then run every command in this lesson in the WSL terminal,
> never in PowerShell. To install PHP 8.4 in Ubuntu:
>
> ```bash
> sudo add-apt-repository ppa:ondrej/php
> sudo apt update
> sudo apt install php8.4-cli php8.4-pgsql php8.4-mbstring php8.4-xml php8.4-curl php8.4-zip unzip
> ```
>
> Then install Composer by following [getcomposer.org/download](https://getcomposer.org/download/).

```bash
php -v
composer -V
laravel --version
docker --version
docker compose version
git --version
```

**Check it works:**

| Command | You need |
|---|---|
| `php -v` | `PHP 8.4.x` |
| `composer -V` | `Composer version 2.x` |
| `laravel --version` | `Laravel Installer 5.x` or newer |
| `docker compose version` | `Docker Compose version v2.x` (note: `docker compose`, with a space) |
| `git --version` | any recent version |

If `laravel` is not found, install it: `composer global require laravel/installer`, then make sure
Composer's global `bin` folder is on your `PATH` (Herd does this for you).

Also check that PHP can talk to PostgreSQL:

```bash
php -m | grep -i pgsql
```

On Windows PowerShell use `php -m | Select-String pgsql`. You must see **`pdo_pgsql`** (and usually
`pgsql`). Herd includes it. If it is missing, see [Troubleshooting](#troubleshooting).

Finally, start **Docker Desktop** and wait until it says it is running. `docker ps` must print a table
header, not an error. In WSL, run `docker ps` in the WSL terminal. If `docker` is not found
there, see the WSL box in [Step 3](#step-3--postgresql-in-docker).

---

## Step 1 — Create the project

**Why:** the installer creates a complete, working Laravel skeleton with the right folder structure,
a `.env` file and a generated `APP_KEY`.

Go to the folder where you keep your projects (with Herd: `~/Herd` so that `billable.test` works
automatically) and run:

```bash
laravel new billable
```

The installer asks questions. Answer like this:

| Question | Answer | Why |
|---|---|---|
| Which starter kit would you like to install? | **None** | Starter kits add authentication and a JavaScript front end. Billable has neither (see PROJECT.md "Not in scope"). |
| Which testing framework do you prefer? | **PHPUnit** | Both tracks use the xUnit style; lesson 11 builds on PHPUnit. |
| Which database will your application use? | **PostgreSQL** | Same database as the Spring track and as production systems. |
| Would you like to run the default database migrations? | **No** | The database is not running yet. We run them in Step 5. |
| Would you like to run `npm install` and `npm run build`? | **No** | We load Pico.css from a CDN; no JavaScript build needed. |

The exact questions change a little between installer versions. If it asks about anything else (for
example AI tooling or initialising Git), answer **No** — we do Git ourselves in Step 8.

```bash
cd billable
php artisan --version
```

**What happens:**

1. The installer downloads the `laravel/laravel` skeleton and runs `composer install`, which fills
   `vendor/` with the framework and all libraries listed in `composer.json`.
2. It copies `.env.example` to `.env` and runs `php artisan key:generate`, which writes a random
   `APP_KEY` into `.env`. Laravel uses this key to encrypt cookies and sessions.
3. Because you chose PostgreSQL, it sets `DB_CONNECTION=pgsql` in `.env`.

**Check it works:** `php artisan --version` prints `Laravel Framework 12.x.x` (or a newer major).

---

## Step 2 — A tour of the project

**Why:** you will open these files hundreds of times. Five minutes now saves confusion later.

```
billable/
├── app/                       ← YOUR code (namespace App\)
│   ├── Http/Controllers/      ← controllers (Controller.php is the empty base class)
│   ├── Models/                ← Eloquent models (User.php exists; ours arrive in lesson 03)
│   └── Providers/             ← AppServiceProvider: framework set-up code
├── bootstrap/app.php          ← creates the application; routing + middleware config
├── config/                    ← config files; most values come from .env
├── database/
│   ├── migrations/            ← database schema changes, in order
│   ├── factories/ seeders/    ← test data (lesson 03)
├── public/index.php           ← THE FRONT CONTROLLER — every request starts here
├── resources/views/           ← Blade templates
├── routes/web.php             ← URL → controller mapping for web pages
├── storage/                   ← logs, compiled views, cache (not committed)
├── tests/                     ← Unit/ and Feature/ tests
├── vendor/                    ← installed libraries (NOT committed, never edit)
├── .env                       ← your local settings and secrets (NOT committed)
├── .env.example               ← template for .env (committed)
├── artisan                    ← Laravel's command-line tool
└── composer.json / .lock      ← dependency list and exact installed versions
```

Open `public/index.php`. It is short. The important lines:

```php
// public/index.php (generated — do not change)
define('LARAVEL_START', microtime(true));

require __DIR__.'/../vendor/autoload.php';

$app = require_once __DIR__.'/../bootstrap/app.php';

$app->handleRequest(Request::capture());
```

**What happens** on every request:

1. `LARAVEL_START` records the start time (we use it in the Advanced task).
2. `vendor/autoload.php` registers Composer's autoloader: when code says `new HomeController`, PHP
   finds the file by its namespace.
3. `bootstrap/app.php` builds the application object — the service container — and tells it where the
   routes are.
4. `Request::capture()` builds a `Request` object from PHP's globals (`$_GET`, `$_POST`, headers).
   `handleRequest()` sends it through the HTTP kernel, middleware and router, and sends the response.

Now open `bootstrap/app.php`:

```php
// bootstrap/app.php (generated)
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware): void {
        //
    })
    ->withExceptions(function (Exceptions $exceptions): void {
        //
    })->create();
```

This is where routes and middleware are registered. **Version pitfall:** tutorials for Laravel 10 and
older talk about `app/Http/Kernel.php` and `RouteServiceProvider`. Those files do not exist any more
(since Laravel 11). Their job moved into `bootstrap/app.php`.

---

## Step 3 — PostgreSQL in Docker

**Why:** everybody gets exactly the same database version, with no installation on your laptop, and
you can reset it in seconds.

> **Windows with WSL — read this before you start.** The commands below are identical in WSL, but
> three things go wrong:
>
> 1. **Wrong folder.** Your project must be in the Linux file system, e.g. `~/projects/billable`. Run
>    `pwd`. If the path starts with `/mnt/c/`, the project is on the Windows drive. That works, but it
>    is slow and breaks file watching. Move the project to a folder under `~`.
> 2. **No `docker` in WSL.** You do **not** install Docker inside Ubuntu. Docker Desktop runs on
>    Windows and gives the `docker` command to your WSL distro. If `docker ps` in the WSL terminal says
>    `command not found` or `could not be found in this WSL 2 distro`, open Docker Desktop → Settings →
>    Resources → **WSL integration**, switch on your distro (e.g. Ubuntu), click *Apply & restart*, then
>    open a new WSL terminal. Do not `apt install docker.io` as well. Two Dockers fight each other.
> 3. **A second PostgreSQL on port 5432.** If PostgreSQL was ever installed with `apt` inside WSL, or
>    on Windows itself, it may answer on port 5432 instead of the container. You then get
>    `password authentication failed for user "billable"` even though the password is right. Stop it:
>    `sudo service postgresql stop` in WSL (and `sudo systemctl disable postgresql` so it stays off),
>    or on Windows open *Services*, find `postgresql-x64-…` and set it to *Manual* and *Stopped*.
>
> **Where is the database?** The container runs inside Docker Desktop, not in your Ubuntu. Docker
> Desktop forwards port 5432 to both Windows and WSL, so `127.0.0.1:5432` works from the WSL terminal
> (where Laravel runs) and from Windows tools such as DBeaver or DataGrip. You never need the WSL IP
> address.

Create `compose.yaml` in the project root (the same folder as `artisan`):

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

Start it:

```bash
docker compose up -d
docker compose ps
```

**What happens:**

1. `image: postgres:17` — Docker downloads the official PostgreSQL image, major version 17. We pin the
   major version: with `latest` you would silently get version 18 one day, which stores its data in a
   different folder and would not start with this file.
2. `environment` — on the **first** start, the image creates the database `billable` and the user
   `billable` with password `secret`. On later starts these values are ignored because the data
   already exists.
3. `ports: "5432:5432"` — port 5432 on your laptop is forwarded to port 5432 in the container, so
   Laravel can connect to `127.0.0.1:5432`.
4. `volumes` — the data lives in a named Docker volume `pgdata`. It survives `docker compose down`.
   `docker compose down -v` deletes it (a completely fresh database).
5. `healthcheck` — Docker runs `pg_isready` every 5 seconds, so `docker compose ps` can tell you when
   the database actually accepts connections.
6. `-d` means "detached": the container runs in the background.

**Check it works:** `docker compose ps` shows the `postgres` service with status `Up … (healthy)`. You
can open a SQL prompt inside the container:

```bash
docker compose exec postgres psql -U billable -d billable -c "select version();"
```

You see a line starting with `PostgreSQL 17.`

---

## Step 4 — Connect Laravel to the database

**Why:** Laravel reads connection settings from `.env`. They must match `compose.yaml` exactly.

Open `.env` and find the `DB_` lines. The installer may have left some of them commented out with `#`.
Make the block look like this:

```dotenv
# .env
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=billable
DB_USERNAME=billable
DB_PASSWORD=secret
```

Make the same change in `.env.example`, but that file is committed, so it shows the *shape* of the
configuration to other developers. Because `secret` is only our local Docker password, it is fine to
leave it there. A real production password never goes into `.env.example`.

While you are in `.env`, also set the application name:

```dotenv
# .env
APP_NAME=Billable
```

**Timezone.** Billable works in Estonian time. Open `config/app.php` and find the `'timezone'` line.

```php
// config/app.php — find this line and change the value
'timezone' => 'Europe/Tallinn',
```

If your version has `'timezone' => env('APP_TIMEZONE', 'UTC')`, set `APP_TIMEZONE=Europe/Tallinn` in
`.env` (and `.env.example`) instead of editing the PHP file.

**What happens:**

1. `config/database.php` contains a `pgsql` connection that reads `env('DB_HOST')`,
   `env('DB_DATABASE')` and so on. You configure through `.env`; you do not edit `config/database.php`.
2. The timezone setting controls what `now()` and `today()` return. Without it, the date would be in
   UTC — shortly after midnight in Tallinn, the page would still show yesterday.

**Check it works:**

```bash
php artisan about
```

In the **Environment** section, `Timezone` shows `Europe/Tallinn`. In the **Drivers** section,
`Database` shows `pgsql`.

---

## Step 5 — Run the default migrations

**Why:** Laravel stores sessions, cache and queued jobs in the database by default. Those tables must
exist before the first page load, or you get an error about a missing `sessions` table.

```bash
php artisan migrate
```

**What happens:**

1. Laravel connects to PostgreSQL using the `.env` values. If it cannot connect, you get an error now
   rather than later — this command is also our connection test.
2. It creates a `migrations` table that remembers which migration files have already run.
3. It runs the three files in `database/migrations/`: `create_users_table` (tables `users`,
   `password_reset_tokens`, `sessions`), `create_cache_table` and `create_jobs_table`.

Billable is single-user, so we will not use the `users` table. We keep it anyway: it is part of the
framework's defaults and deleting it gives us nothing.

**Check it works:** the output lists three migrations with `DONE`. Then:

```bash
php artisan migrate:status
docker compose exec postgres psql -U billable -d billable -c "\dt"
```

`\dt` lists tables such as `cache`, `jobs`, `migrations`, `sessions`, `users`.

**Commit:** we do not have Git yet — we create the repository in Step 8 and commit then.

---

## Step 6 — The layout

**Why:** every page in Billable shares the same `<head>`, navigation and styling. A layout means we
write that HTML once. If we copied it into every view, a change to the menu would mean editing 20 files.

Create `resources/views/layouts/app.blade.php`:

```blade
{{-- resources/views/layouts/app.blade.php --}}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>@yield('title', 'Home') · Billable</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@picocss/pico@2/css/pico.min.css">
</head>
<body>
    <header class="container">
        <nav>
            <ul>
                <li><strong>Billable</strong></li>
            </ul>
            <ul>
                <li><a href="{{ route('home') }}">Home</a></li>
            </ul>
        </nav>
    </header>

    <main class="container">
        @if (session('status'))
            <article role="status">{{ session('status') }}</article>
        @endif

        @yield('content')
    </main>

    <footer class="container">
        <small>Billable</small>
    </footer>
</body>
</html>
```

**What happens:**

1. `@yield('title', 'Home')` is a placeholder. A page that uses this layout can fill it with
   `@section('title', 'Clients')`. If it does not, the default `Home` is used.
2. The `<link>` loads Pico.css from the jsDelivr CDN. Pico styles plain HTML elements (`nav`,
   `article`, `table`, `form`) without any CSS classes, so our templates stay clean. `class="container"`
   centres the content with a maximum width.
3. `{{ route('home') }}` generates the URL of the route **named** `home`. We create that route in the
   next step. If the URL of the home page changes one day, the link still works.
4. `session('status')` shows a one-time "flash" message, such as "Client saved." We do not send any
   flash messages yet — lesson 05 does — but the layout is ready for them.
5. `@yield('content')` is where each page puts its main content.
6. `{{ ... }}` always escapes HTML. If a client's name contains `<script>`, it is shown as text, not
   executed. This is one of those "solved problems" a framework gives you.

**Another way.** Many Laravel 12 projects write the layout as a **Blade component** instead: a file
`resources/views/components/layout.blade.php` that prints `{{ $slot }}`, used as
`<x-layout>…</x-layout>` (the official starter kits call theirs `<x-layouts.app>`).
Both styles work. This course uses `@extends` and `@section` because each placeholder has a visible
name, and every later view follows the same pattern.

---

## Step 7 — HomeController, the home view and the route

**Why:** this is our first full MVC request. The **controller** decides what data the page needs
(today's date), the **view** only displays it.

Create the controller with Artisan:

```bash
php artisan make:controller HomeController
```

Replace the content of the generated file:

```php
<?php

// app/Http/Controllers/HomeController.php

namespace App\Http\Controllers;

use Illuminate\View\View;

class HomeController extends Controller
{
    public function index(): View
    {
        return view('home', [
            'today' => today(),
        ]);
    }
}
```

Create the view `resources/views/home.blade.php`:

```blade
{{-- resources/views/home.blade.php --}}
@extends('layouts.app')

@section('title', 'Home')

@section('content')
    <hgroup>
        <h1>Billable</h1>
        <p>Time tracking and invoicing for freelancers.</p>
    </hgroup>

    <p>Today is {{ $today->format('d.m.Y') }}.</p>
@endsection
```

Now open `routes/web.php`. It contains a route that returns the default `welcome` view. Replace the
whole file:

```php
<?php

// routes/web.php

use App\Http\Controllers\HomeController;
use Illuminate\Support\Facades\Route;

Route::get('/', [HomeController::class, 'index'])->name('home');
```

Delete the default welcome page, we do not need it any more:

```bash
rm resources/views/welcome.blade.php
```

(On Windows PowerShell: `Remove-Item resources/views/welcome.blade.php`.)

**What happens:**

1. `Route::get('/', …)` tells the router: a `GET` request for the path `/` must be handled by the
   `index` method of `HomeController`. `[HomeController::class, 'index']` is an array of class name and
   method name — the router creates the controller object and calls the method for you (inversion of
   control).
2. `->name('home')` gives the route a name, used by `route('home')` in the layout.
3. `today()` is a Laravel helper that returns a `Carbon` date object for today at 00:00, in the
   application timezone (`Europe/Tallinn`). Carbon is a date library that extends PHP's `DateTime`.
4. `view('home', [...])` finds `resources/views/home.blade.php` (the dot notation `layouts.app` means
   the folder `layouts/`, file `app.blade.php`) and makes `$today` available in it.
5. The return type `View` documents what the method returns. The controller does not produce HTML
   itself.
6. In the view, `@extends('layouts.app')` says "render me inside the layout". `@section('content')` …
   `@endsection` fills the `@yield('content')` placeholder.
7. `$today->format('d.m.Y')` formats the date the Estonian way: `30.09.2026`. Formatting for display is
   the view's job; *choosing* the date is the controller's job.

Why does the controller supply the date instead of the view calling `today()` itself? Because the view
should not make decisions. In lesson 12 you will replace "now" with a fixed time in tests. That is only
easy if "now" enters the page in one known place.

**Check it works:**

```bash
php artisan route:list
```

You see a line like:

```
GET|HEAD   / ............................ home › HomeController@index
```

and a few framework routes such as `up` (the health check). `HEAD` is added automatically for every
`GET` route.

---

## Step 8 — Run it and watch the request

Start the development server:

```bash
php artisan serve
```

Open `http://127.0.0.1:8000`. With Herd you can instead open `http://billable.test` (Herd serves
every project in its folder without `artisan serve`).

**Check it works:** you see a styled page with a nav bar, the heading **Billable**, and
"Today is …" with today's date in `dd.mm.yyyy` format.

Now look at the request itself:

1. Open the browser's developer tools (F12) → **Network** tab → reload the page.
2. Click the first request (`/` or `localhost`). Under **Headers** you see: Request Method `GET`,
   Status Code `200 OK`, response header `Content-Type: text/html; charset=utf-8`, and `Set-Cookie`
   headers for `laravel_session` and `XSRF-TOKEN`.
3. Where did those cookies come from? Not from your code. The **middleware** in the `web` group
   (`StartSession`, `EncryptCookies`, CSRF protection) added them on the way out. That is the lifecycle
   in action.
4. Visit `http://127.0.0.1:8000/nothing-here`. You get a **404** page — the router found no matching
   route.
5. Visit `http://127.0.0.1:8000/up`. This is Laravel's built-in health-check route. Monitoring tools
   use it to check that the app is alive.

The terminal where `artisan serve` runs prints one line per request, for example
`2026-09-30 18:02:11 / ........ ~ 0.2ms`.

Stop the server with **Ctrl+C** when you need the terminal.

---

## Step 9 — Git and GitHub

**Why:** your Git history is evidence for ÕV7 and ÕV9. It starts today.

First check whether the installer already created a repository:

```bash
git status
```

If it says `fatal: not a git repository`, create one:

```bash
git init -b main
```

Before the first commit, **check that secrets are ignored**:

```bash
git status --short
git check-ignore -v .env
```

**Check it works:** `git check-ignore -v .env` prints something like `.gitignore:5:.env  .env`, which
means Git ignores `.env`. `git status --short` must **not** list `.env`, `vendor/` or `node_modules/`.
It should list `compose.yaml`, `app/`, `routes/`, `resources/` and so on.

Laravel's `.gitignore` already contains `/vendor`, `.env`, `/node_modules` and other entries. Open it
and read it once, so you know what is excluded.

Now commit. Which command you use depends on what `git log` shows.

**Case A — you just ran `git init` (no commits yet):** one commit with everything.

```bash
git add .
git commit -m "chore: create Laravel project with PostgreSQL and home page"
```

**Case B — the installer already made an initial commit** (`git log --oneline` shows one line):
commit only your own changes on top of it.

```bash
git add .
git commit -m "feat(home): add PostgreSQL set-up, layout and home page"
```

The message format `type(scope): summary` is **Conventional Commits** — lesson 02 explains it. From
the next lesson on we commit after each step instead of once at the end.

**Push to GitHub:**

1. On github.com click **New repository**. Name: `billable`. Do **not** tick "Add a README",
   ".gitignore" or "license" — the repository must be empty, otherwise the first push is rejected.
2. Copy the commands GitHub shows under "…or push an existing repository from the command line". They
   look like this (with your user name):

```bash
git remote add origin git@github.com:<your-user>/billable.git
git branch -M main
git push -u origin main
```

If you have not set up SSH keys, use the HTTPS URL (`https://github.com/<your-user>/billable.git`)
instead; Git will ask you to log in through the browser.

**Check it works:** refresh the repository page on GitHub. You see your files, and `.env` is **not**
among them. Share the repository with the teacher (Settings → Collaborators) if it is private.

---

## Troubleshooting

| Error / symptom | Cause | Fix |
|---|---|---|
| `could not find driver (Connection: pgsql, …)` | PHP has no `pdo_pgsql` extension | Herd: it is included — make sure the terminal uses Herd's PHP (`which php` / `where php`). Other installs: enable `extension=pdo_pgsql` in `php.ini` (find it with `php --ini`) or install the package (`sudo apt install php8.4-pgsql` on Ubuntu). Restart the terminal. |
| `SQLSTATE[08006] … Connection refused` | PostgreSQL is not running or not on port 5432 | Start Docker Desktop, run `docker compose up -d`, wait for `(healthy)` in `docker compose ps`. |
| `password authentication failed for user "billable"` | The volume was created earlier with other credentials (the env vars only apply on the first start), or `.env` has a typo | Check `.env`. If it is correct, reset the database: `docker compose down -v` then `docker compose up -d`. |
| `Bind for 0.0.0.0:5432 failed: port is already allocated` | Another PostgreSQL (local installation or another project's container) uses port 5432 | Stop the other one (`docker ps` to find containers; stop a local PostgreSQL service in Services / `brew services stop postgresql`). Or map another port: `"5433:5432"` in `compose.yaml` and `DB_PORT=5433` in `.env`. |
| WSL: `The command 'docker' could not be found in this WSL 2 distro` | Docker Desktop's WSL integration is off for your distro | Docker Desktop → Settings → Resources → WSL integration → switch on your distro → *Apply & restart*. Open a new WSL terminal. |
| WSL: `password authentication failed` although `.env` is correct and `down -v` did not help | Another PostgreSQL (installed with `apt` in WSL, or on Windows) answers on port 5432 | `sudo service postgresql stop` in WSL, or stop the `postgresql-x64-…` service in Windows *Services*. See the WSL box in Step 3. |
| WSL: everything is very slow, or file changes are not picked up | The project is under `/mnt/c/…` (the Windows drive) | Move it into the Linux file system, e.g. `~/projects/billable`. |
| WSL: `php` / `composer` not found, though Herd is installed | Herd runs on Windows only and is not visible in WSL | Install PHP 8.4 and Composer inside Ubuntu (Step 0). |
| `Cannot connect to the Docker daemon` / `error during connect` | Docker Desktop is not running | Start Docker Desktop and wait until it is ready. |
| `relation "sessions" does not exist` on the first page load | Migrations have not run | `php artisan migrate`. |
| `Route [home] not defined.` | The layout uses `route('home')` but the route has no name, or the name is misspelled | Add `->name('home')` in `routes/web.php`. Run `php artisan route:list`. |
| `View [home] not found.` | File name or folder wrong | The file must be `resources/views/home.blade.php` — note `.blade.php`. |
| The date is one day off near midnight | Timezone still UTC | Step 4: set the timezone, then `php artisan config:clear`. |
| Changes in `.env` have no effect | Configuration is cached | `php artisan config:clear`. Never run `config:cache` in development. |
| `laravel: command not found` | Composer's global bin folder is not on `PATH` | Run `composer global config bin-dir --absolute` and add that folder to `PATH`, or use Herd. |
| `Failed to listen on 127.0.0.1:8000` | Another `artisan serve` is still running | Close the other terminal, or `php artisan serve --port=8001`. |

---

## Recap

- `compose.yaml` — PostgreSQL 17 in Docker, same settings in both tracks.
- `.env` / `.env.example` — database connection, app name; `config/app.php` — timezone.
- `php artisan migrate` — created Laravel's default tables and proved the connection works.
- `resources/views/layouts/app.blade.php` — shared layout with Pico.css, `@yield` placeholders.
- `app/Http/Controllers/HomeController.php` — the C: chooses the data (today's date).
- `resources/views/home.blade.php` — the V: displays it, inside the layout.
- `routes/web.php` — maps `GET /` to `HomeController@index`, named `home`.
- A request travels: `public/index.php` → `bootstrap/app.php` → kernel → middleware → router →
  controller → view → response back through middleware.

---

## Independent work — solutions

> **The tasks are not here.** The task descriptions (Basic / Intermediate / Advanced, with acceptance
> criteria) are in this lesson's [README.md → Independent work (~2 h)](README.md#independent-work-2-h).
> Read them there first and build the features in your project. This section only contains the
> solutions — try each task yourself, then open a solution to compare or when you are stuck.

<details>
<summary>Task 1 — About page (Basic)</summary>

Add a second method to the existing controller. Both pages are simple, informational pages, so they
belong together.

```php
<?php

// app/Http/Controllers/HomeController.php

namespace App\Http\Controllers;

use Illuminate\View\View;

class HomeController extends Controller
{
    public function index(): View
    {
        return view('home', [
            'today' => today(),
        ]);
    }

    public function about(): View
    {
        return view('about');
    }
}
```

```blade
{{-- resources/views/about.blade.php --}}
@extends('layouts.app')

@section('title', 'About')

@section('content')
    <h1>About Billable</h1>

    <p>
        Billable is a time-tracking and invoicing tool for freelancers and small agencies.
        You log the hours you work for your clients, and Billable turns them into correct
        invoices with VAT.
    </p>
    <p>
        It is built step by step in the M8 module at Kuressaare Ametikool.
    </p>
@endsection
```

Add the route under the home route:

```php
// routes/web.php — add below the home route
Route::get('/about', [HomeController::class, 'about'])->name('about');
```

Check: `http://127.0.0.1:8000/about` shows the page, the tab title is "About · Billable", and
`php artisan route:list` lists `about`.

```bash
git add .
git commit -m "feat(home): add about page"
```

</details>

<details>
<summary>Task 2 — Trace a request (Basic)</summary>

Your text must be in your own words — this is an example of the level of detail, not something to copy.

```markdown
<!-- docs/request-lifecycle.md -->
# Request lifecycle: GET /about

This is what happens when the browser asks for `http://127.0.0.1:8000/about`.

1. **Web server.** `php artisan serve` receives the request. `/about` is not a file in `public/`,
   so the request is passed to `public/index.php`.
2. **Front controller — `public/index.php`.** Records `LARAVEL_START`, loads `vendor/autoload.php`,
   loads `bootstrap/app.php`, builds a `Request` object with `Request::capture()` and calls
   `handleRequest()`.
3. **Application — `bootstrap/app.php`.** Creates the application (service container) and registers
   `routes/web.php` as the web routes.
4. **HTTP kernel — `Illuminate\Foundation\Http\Kernel`.** Sends the request through global middleware
   and the `web` middleware group (for example `StartSession`, `ValidateCsrfToken`).
5. **Router — `routes/web.php`.** Finds `GET /about` → `HomeController@about` (route name `about`).
6. **Controller — `app/Http/Controllers/HomeController.php`.** `about()` returns `view('about')`.
7. **View — `resources/views/about.blade.php`** extends **`resources/views/layouts/app.blade.php`**.
   Blade compiles both to PHP in `storage/framework/views/` and renders HTML.
8. **Response.** The HTML becomes a `Response` object, goes back through the middleware (which adds the
   session cookie) and is sent to the browser with status 200.

Evidence (`php artisan route:list`):

    GET|HEAD   / ........................ home › HomeController@index
    GET|HEAD   about .................. about › HomeController@about

MVC in this request: `HomeController` is the controller, `about.blade.php` is the view. There is no
model yet, because the page shows only fixed text.
```

To see the middleware list yourself: `php artisan route:list -v` shows the middleware for each route.

```bash
git add docs/request-lifecycle.md
git commit -m "docs: describe request lifecycle"
```

</details>

<details>
<summary>Task 3 — Named routes and an active nav link (Intermediate)</summary>

The routes are already named (`home`, `about`) if you did Task 1. Now use the names in the layout and
mark the current page. `request()->routeIs('about')` returns `true` when the current route has that
name.

Replace the second `<ul>` of the `<nav>` in `resources/views/layouts/app.blade.php`:

```blade
{{-- resources/views/layouts/app.blade.php — the second <ul> inside <nav> --}}
<ul>
    <li>
        <a href="{{ route('home') }}" @if (request()->routeIs('home')) aria-current="page" @endif>Home</a>
    </li>
    <li>
        <a href="{{ route('about') }}" @if (request()->routeIs('about')) aria-current="page" @endif>About</a>
    </li>
</ul>
```

Prove that named routes protect your links: change the URL in `routes/web.php` to
`Route::get('/about-billable', …)->name('about')`, reload — the nav link still works and points to the
new URL. Change it back afterwards (or keep it, your choice).

Why `aria-current`? It tells screen readers which page is active, and Pico.css styles it, so we get
accessibility and styling from one attribute.

```bash
git commit -am "feat(home): add about link with active state to navigation"
```

</details>

<details>
<summary>Task 4 — X-Request-Time middleware (Advanced)</summary>

```bash
php artisan make:middleware AddRequestTime
```

```php
<?php

// app/Http/Middleware/AddRequestTime.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Symfony\Component\HttpFoundation\Response;

class AddRequestTime
{
    /**
     * Measures the time from the front controller until the response is ready.
     */
    public function handle(Request $request, Closure $next): Response
    {
        // LARAVEL_START is defined in public/index.php. It is missing in tests and
        // console commands, so we fall back to "now".
        $start = defined('LARAVEL_START') ? LARAVEL_START : microtime(true);

        $response = $next($request);

        $milliseconds = (microtime(true) - $start) * 1000;
        $response->headers->set('X-Request-Time', sprintf('%.1fms', $milliseconds));

        Log::info(sprintf(
            '%s /%s -> %d in %.1f ms',
            $request->method(),
            ltrim($request->path(), '/'),
            $response->getStatusCode(),
            $milliseconds,
        ));

        return $response;
    }
}
```

Register it as **global** middleware in `bootstrap/app.php`. Replace the empty `withMiddleware`
closure:

```php
// bootstrap/app.php — replace the withMiddleware(...) call
->withMiddleware(function (Middleware $middleware): void {
    $middleware->append(\App\Http\Middleware\AddRequestTime::class);
})
```

(You can also add `use App\Http\Middleware\AddRequestTime;` at the top of the file and write
`AddRequestTime::class`.)

**What happens:**

1. Everything **before** `$next($request)` runs on the way **in**; everything **after** runs on the way
   **out**. `$next` passes the request to the next middleware, and finally to the router and your
   controller. This is the "onion" model: your middleware wraps the rest of the lifecycle.
2. `LARAVEL_START` was recorded by `public/index.php` — the first line of the lifecycle — so the number
   includes framework boot time, not only the controller.
3. `append()` adds the middleware to the global stack, so it runs for every request, including 404s.
   Using a float here is fine: this is a duration for humans, not money (R1 is about money).

**Check it works:**

```bash
curl -I http://127.0.0.1:8000/
```

The output contains a line like `X-Request-Time: 18.4ms`. `storage/logs/laravel.log` contains lines
like `local.INFO: GET / -> 200 in 18.4 ms`. In the browser's Network tab you find the header under
Response Headers.

```bash
git add .
git commit -m "feat(common): add request time header middleware"
```

</details>
