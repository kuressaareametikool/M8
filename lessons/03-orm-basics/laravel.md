# 03 — ORM basics: models and migrations · Laravel

[← Concepts](README.md) · Starting point: end of lesson 02 · Estimated time in class: 2 h · Independent work tasks: [README](README.md#independent-work-2-h)

## What we build today

- `App\Enums\BillingType` — a backed string enum with a `label()` method
- migrations for `clients` and `projects`, exactly as in [PROJECT.md](../../PROJECT.md#data-model)
- `App\Models\Client` and `App\Models\Project` with a one-to-many relationship
- `ClientFactory` and `ProjectFactory`, and a `DatabaseSeeder` that creates the fixed Billable test data
- a Tinker session where we run our first queries and look at the SQL
- the home page shows "3 clients · 7 projects"

**Final result:** `php artisan migrate:fresh --seed` builds the whole database from nothing, and the home
page at `http://127.0.0.1:8000` (or `https://billable.test` with Herd) shows the counts from the database.

---

## Step 1 — Check that Laravel can reach PostgreSQL

**Why:** everything today talks to the database. If the connection is wrong, find out now, not in the
middle of step 5.

Start Docker Desktop, then start the database from the project root:

```bash
docker compose up -d
```

Check your `.env`. From lesson 01 it should contain:

```ini
# .env
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=billable
DB_USERNAME=billable
DB_PASSWORD=secret
```

Now ask Laravel which migrations it knows:

```bash
php artisan migrate:status
```

**What happens:**

1. Artisan reads `.env` and connects to PostgreSQL on port 5432.
2. It reads the `migrations` table, where Laravel records every migration that has already run.
3. It compares that list with the files in `database/migrations/` and prints the status of each.

**Check it works:** you see the three migrations that come with a new Laravel project
(`create_users_table`, `create_cache_table`, `create_jobs_table`) marked `Ran`. Billable has no login,
but we leave these tables alone — Laravel's cache and queue features use them.

If you see `Connection refused`, Docker is not running — see [Troubleshooting](#troubleshooting).

---

## Step 2 — The `BillingType` enum

**Why:** a project is billed in one of three ways. We never want the text `"hourly"` spread through the
code as a loose string — a typo like `"hourley"` would be a silent bug. A **backed enum** is a PHP type
with a fixed set of cases, each with a string value that goes into the database.

Create the folder `app/Enums` and the file:

```php
<?php

// app/Enums/BillingType.php

namespace App\Enums;

enum BillingType: string
{
    case Hourly = 'hourly';
    case Fixed = 'fixed';
    case Capped = 'capped';

    /**
     * Human-readable name for pages and forms.
     */
    public function label(): string
    {
        return match ($this) {
            self::Hourly => 'Hourly',
            self::Fixed => 'Fixed price',
            self::Capped => 'Capped hourly',
        };
    }
}
```

**What happens:**

1. `enum BillingType: string` declares a **backed** enum: every case has a string value.
2. `case Hourly = 'hourly'` — in code you write `BillingType::Hourly`; in the database the value is
   `hourly`. The lowercase values are the same as in the Spring track and in PROJECT.md.
3. `label()` is a normal method. `match ($this)` returns the text for the current case. Because `match`
   has no `default`, PHP throws an error if someone adds a fourth case and forgets its label — a mistake
   you notice immediately instead of seeing an empty label.
4. PHP gives every backed enum two static methods for free: `BillingType::from('fixed')` returns the
   case (or throws a `ValueError` for an unknown value), and `BillingType::tryFrom('x')` returns `null`
   instead of throwing.

**Check it works:** nothing to run yet. We use the enum in step 6.

---

## Step 3 — `Client`: model, migration and factory in one command

**Why:** a model, its migration and its factory belong together. Artisan can create all three at once.

```bash
php artisan make:model Client -mf
```

**What happens:**

1. `-m` creates a migration file `database/migrations/YYYY_MM_DD_HHMMSS_create_clients_table.php`.
   Artisan guessed the table name `clients` from the model name `Client` (plural, snake_case).
2. `-f` creates `database/factories/ClientFactory.php`.
3. The model is created in `app/Models/Client.php`.

The timestamp at the start of the migration's file name decides the order in which migrations run.
Because we create `clients` before `projects`, the `clients` table will exist when `projects` needs to
point a foreign key at it.

Open the migration and replace its content:

```php
<?php

// database/migrations/YYYY_MM_DD_HHMMSS_create_clients_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('clients', function (Blueprint $table) {
            $table->id();
            $table->string('name', 120);
            $table->string('email')->unique();
            $table->string('vat_number', 20)->nullable();
            $table->timestamp('created_at');
            $table->timestamp('updated_at');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('clients');
    }
};
```

**What happens:**

1. `return new class extends Migration` — the migration is an *anonymous class*. Laravel loads the file
   and gets the object back. There is no class name that could clash with another migration.
2. `up()` runs when you migrate; `down()` runs when you roll back. `down()` must undo exactly what `up()`
   did — here, drop the table.
3. `$table->id()` creates `id bigint` with an auto-increment sequence, as the primary key.
4. `$table->string('name', 120)` creates `varchar(120) not null`. Columns are `NOT NULL` unless you add
   `->nullable()`. The length 120 comes from PROJECT.md; lesson 05's validation will use the same limit.
5. `$table->string('email')->unique()` — `varchar(255)` plus a **unique index**. Now the database itself
   refuses two clients with the same email, even if a validation rule is forgotten.
6. `vat_number` is `->nullable()`, because not every client has a VAT number.
7. `created_at` and `updated_at` are two `timestamp` columns. Eloquent fills them automatically: both on
   insert, `updated_at` on every update. They are `NOT NULL` like every other column without
   `->nullable()`, the same as PROJECT.md and the Spring track's Flyway script.
   You will often see the shortcut `$table->timestamps()` instead. It creates the same two columns but
   makes them **nullable**, so a row inserted past Eloquent (`DB::table(...)->insert()` or raw SQL)
   could have no dates and the database would not complain. We write the two columns out, so the
   database refuses such a row. Every later table with timestamps uses the same two lines.
   Why not add `->useCurrent()` as a database default? `CURRENT_TIMESTAMP` uses the database
   session's time zone (UTC in the Docker image), while Eloquent writes Europe/Tallinn wall-clock time
   (lesson 01). A row filled by the default would be two or three hours off. Without a default, an
   insert that forgets the dates fails loudly instead.

Run it:

```bash
php artisan migrate
```

**Check it works:** the output lists `create_clients_table ... DONE`. Then look at the real table:

```bash
php artisan db:table clients
```

You see the columns `id`, `name` (varchar(120)), `email`, `vat_number` (nullable), `created_at`,
`updated_at`, and the index `clients_email_unique`.

---

## Step 4 — `Project`: model, migration and factory

```bash
php artisan make:model Project -mf
```

Replace the migration's content:

```php
<?php

// database/migrations/YYYY_MM_DD_HHMMSS_create_projects_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('projects', function (Blueprint $table) {
            $table->id();
            $table->foreignId('client_id')->constrained()->restrictOnDelete();
            $table->string('name', 120);
            $table->string('billing_type', 20);
            $table->bigInteger('hourly_rate_cents')->nullable();
            $table->bigInteger('fixed_price_cents')->nullable();
            $table->bigInteger('budget_cap_cents')->nullable();
            $table->smallInteger('rounding_minutes')->default(1);
            $table->timestamp('archived_at')->nullable();
            $table->timestamp('created_at');
            $table->timestamp('updated_at');

            $table->index('client_id');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('projects');
    }
};
```

**What happens:**

1. `foreignId('client_id')` creates a `bigint` column — the same type as `clients.id`.
2. `->constrained()` adds a **foreign key constraint**. From the column name `client_id` Laravel guesses
   the target: table `clients`, column `id`. That is why the naming convention matters.
3. `->restrictOnDelete()` means `ON DELETE RESTRICT`: PostgreSQL refuses to delete a client who still
   has projects. PROJECT.md asks for this.
4. `billing_type` is a plain `varchar(20)`. We store the enum's value (`hourly`, `fixed`, `capped`) —
   a string, never a number (see [Concepts §5](README.md#5-mapping-types)).
5. The three money columns are `bigint` (64-bit integers, so large totals never overflow) and
   **nullable**: an hourly project has no fixed price, a fixed project has no hourly rate. Which ones are required depends on the billing type; that is checked by
   validation in lesson 05.
6. `smallInteger('rounding_minutes')->default(1)` — a 2-byte integer with default 1.
7. `timestamp('archived_at')->nullable()` — `NULL` means "active".
8. `$table->index('client_id')` — PostgreSQL does **not** create an index for a foreign key
   automatically. Almost every query on projects filters by client, so we add one now. Lesson 14 talks
   about indexes in depth.

```bash
php artisan migrate
php artisan db:table projects
```

**Check it works:** `db:table projects` lists all 11 columns and a foreign key
`projects_client_id_foreign` → `clients`.

**Commit:** `feat(db): create clients and projects tables`

---

## Step 5 — The `Client` model

Replace `app/Models/Client.php`:

```php
<?php

// app/Models/Client.php

namespace App\Models;

use Database\Factories\ClientFactory;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Client extends Model
{
    /** @use HasFactory<ClientFactory> */
    use HasFactory;

    /**
     * Fields that may be set from outside data (forms) with create() / fill() / update().
     */
    protected $fillable = [
        'name',
        'email',
        'vat_number',
    ];

    /**
     * @return HasMany<Project, $this>
     */
    public function projects(): HasMany
    {
        return $this->hasMany(Project::class);
    }
}
```

**What happens:**

1. `extends Model` makes `Client` an **Active Record**: it has `save()`, `delete()`, `Client::find()`,
   `Client::where()` and many more, inherited from `Model`.
2. Notice what is **not** here: no list of columns. Eloquent reads the columns from each row it loads.
   `$client->name` works because the row has a `name` column. (This is a big difference from JPA, where
   every column is a declared field.)
3. The table name `clients` is guessed from the class name. We do not need `protected $table`.
4. `HasFactory` connects the model to `ClientFactory`, so `Client::factory()` works. The `@use` comment
   helps your editor and static analysis know the factory type.
5. `$fillable` is the mass-assignment allow-list. `Client::create(['name' => ..., 'email' => ...])`
   works; a key that is not in the list (for example `id`) is not assigned.
6. `projects()` returns a `HasMany` **relationship object**. Laravel finds projects by the foreign key
   `client_id` — again guessed from the class name `Client`.

Two ways to use a relationship — learn the difference now, it matters in lesson 06:

| You write | What it is | SQL |
|---|---|---|
| `$client->projects` (property) | A **Collection** of `Project` models, loaded on first access and then cached on the object | `select * from "projects" where "projects"."client_id" = ? and ...` — runs once |
| `$client->projects()` (method) | A **query builder** you can add conditions to | nothing yet; runs when you call `->get()`, `->count()` … |

`$client->projects()->count()` asks the database to count (`select count(*)`). `$client->projects->count()`
loads every project into PHP and counts them there. For 3 projects the difference is nothing; for 30 000
time entries it is huge.

---

## Step 6 — The `Project` model

Replace `app/Models/Project.php`:

```php
<?php

// app/Models/Project.php

namespace App\Models;

use App\Enums\BillingType;
use Database\Factories\ProjectFactory;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Project extends Model
{
    /** @use HasFactory<ProjectFactory> */
    use HasFactory;

    /**
     * client_id is not here on purpose: it comes from the URL (/clients/{client}/projects),
     * never from form fields. archived_at is set by an "archive" action, not by a form.
     */
    protected $fillable = [
        'name',
        'billing_type',
        'hourly_rate_cents',
        'fixed_price_cents',
        'budget_cap_cents',
        'rounding_minutes',
    ];

    /**
     * @return array<string, string>
     */
    protected function casts(): array
    {
        return [
            'billing_type' => BillingType::class,
            'hourly_rate_cents' => 'integer',
            'fixed_price_cents' => 'integer',
            'budget_cap_cents' => 'integer',
            'rounding_minutes' => 'integer',
            'archived_at' => 'datetime',
        ];
    }

    /**
     * @return BelongsTo<Client, $this>
     */
    public function client(): BelongsTo
    {
        return $this->belongsTo(Client::class);
    }

    public function isArchived(): bool
    {
        return $this->archived_at !== null;
    }
}
```

**What happens:**

1. **Casts** tell Eloquent how to convert a column value when you read it and write it:
   - `'billing_type' => BillingType::class` — reading gives you a `BillingType` case, not a string.
     `$project->billing_type === BillingType::Fixed` works, and `$project->billing_type->label()` gives
     `"Fixed price"`. Writing accepts the enum case (or its string value) and stores `fixed`.
   - `'integer'` makes sure the cents are PHP `int`s. With PostgreSQL they already are, but the cast
     documents the intent and protects you if another database driver returns strings.
   - `'archived_at' => 'datetime'` gives a `Carbon` date object, so you can write
     `$project->archived_at->diffForHumans()`. `created_at` and `updated_at` are cast automatically.
2. Since Laravel 11, casts are defined in a `casts()` **method**. Older tutorials (and many AI answers)
   use a `protected $casts = [...]` property. Both still work; we use the method.
3. `client()` is the other side of the relationship. `belongsTo` looks for the column `client_id` on
   *this* table. `$project->client` loads the client.
4. `isArchived()` is a small rule about one object, so it belongs on the model.
5. `$fillable` leaves out `client_id` and `archived_at` — the comment says why. When you create a
   project for a client in lesson 05, you write `$client->projects()->create($data)`, and Eloquent fills
   `client_id` from the relationship.

**Make the protection loud.** By default, a key that is not in `$fillable` is **silently dropped**: no
error, the value just never reaches the database. That hides typos and forgotten fields. Ask Laravel to
throw instead, in development only. Open `app/Providers/AppServiceProvider.php` and add one line to
`boot()`:

```php
// app/Providers/AppServiceProvider.php — add the import at the top
use Illuminate\Database\Eloquent\Model;

public function boot(): void
{
    Model::preventSilentlyDiscardingAttributes(! $this->app->isProduction());
}
```

In production the line does nothing (a dropped field there should not crash a user's request); locally
and in tests it throws a `MassAssignmentException`. In lesson 06 we replace this line with
`Model::shouldBeStrict()`, which switches on this guard and two others at once. We start with one so
you see what it does.

```bash
php artisan tinker --execute="App\Models\Project::make(['name' => 'X', 'archived_at' => now()]);"
```

Output: `Add fillable property [archived_at] to allow mass assignment on [App\Models\Project].`

**Check it works:** the models are used in the next steps. Run the formatter now to catch style issues
early:

```bash
./vendor/bin/pint
```

**Commit:** `feat(clients): add Client and Project models with BillingType enum`

---

## Step 7 — Factories

**Why:** a factory builds a *valid* model with sensible defaults. Tests in lessons 11–14 will create
hundreds of models this way, overriding only what each test cares about.

Replace `database/factories/ClientFactory.php`:

```php
<?php

// database/factories/ClientFactory.php

namespace Database\Factories;

use App\Models\Client;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<Client>
 */
class ClientFactory extends Factory
{
    /**
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        return [
            'name' => fake()->company(),
            'email' => fake()->unique()->companyEmail(),
            'vat_number' => 'EE'.fake()->numerify('#########'),
        ];
    }
}
```

Replace `database/factories/ProjectFactory.php`:

```php
<?php

// database/factories/ProjectFactory.php

namespace Database\Factories;

use App\Enums\BillingType;
use App\Models\Client;
use App\Models\Project;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<Project>
 */
class ProjectFactory extends Factory
{
    /**
     * By default a project is hourly at 60.00 € per hour.
     *
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        return [
            'client_id' => Client::factory(),
            'name' => ucfirst(fake()->words(2, true)),
            'billing_type' => BillingType::Hourly,
            'hourly_rate_cents' => 6000,
            'fixed_price_cents' => null,
            'budget_cap_cents' => null,
            'rounding_minutes' => 1,
            'archived_at' => null,
        ];
    }

    public function hourly(int $rateCents = 6000): static
    {
        return $this->state(fn () => [
            'billing_type' => BillingType::Hourly,
            'hourly_rate_cents' => $rateCents,
            'fixed_price_cents' => null,
            'budget_cap_cents' => null,
        ]);
    }

    public function fixed(int $priceCents = 100000): static
    {
        return $this->state(fn () => [
            'billing_type' => BillingType::Fixed,
            'hourly_rate_cents' => null,
            'fixed_price_cents' => $priceCents,
            'budget_cap_cents' => null,
        ]);
    }

    public function capped(int $rateCents = 6000, int $capCents = 200000): static
    {
        return $this->state(fn () => [
            'billing_type' => BillingType::Capped,
            'hourly_rate_cents' => $rateCents,
            'fixed_price_cents' => null,
            'budget_cap_cents' => $capCents,
        ]);
    }

    public function archived(): static
    {
        return $this->state(fn () => [
            'archived_at' => now()->subMonths(3),
        ]);
    }
}
```

**What happens:**

1. `definition()` returns the default attributes. `fake()` is the Faker library: `company()` gives a
   random company name, `unique()->companyEmail()` a random email that does not repeat in this run.
2. `'client_id' => Client::factory()` — if you create a project without saying which client, the factory
   creates a new client first and uses its id.
3. The **states** `hourly()`, `fixed()`, `capped()` and `archived()` change a group of attributes
   together. Each state sets *all three* money columns, so a fixed project never accidentally keeps an
   hourly rate from the default. A state is how a factory keeps the rules of the domain.
4. Factories ignore `$fillable` — they are trusted code, not user input — so `archived_at` can be set here.

---

## Step 8 — The seeder

**Why:** everybody in class — and the practice queries in the README — must have **the same** data.
So the seeder uses fixed names and amounts; the factory only fills in the rest.

Replace `database/seeders/DatabaseSeeder.php` completely. The default file creates a test user; Billable
has no login, so we remove it.

```php
<?php

// database/seeders/DatabaseSeeder.php

namespace Database\Seeders;

use App\Models\Client;
use App\Models\Project;
use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    /**
     * Fixed development data. The practice queries in lesson 03 depend on these exact values.
     */
    public function run(): void
    {
        $acme = Client::factory()->create([
            'name' => 'Acme OÜ',
            'email' => 'billing@acme.test',
            'vat_number' => 'EE101234567',
        ]);

        Project::factory()->for($acme)->hourly(6500)->create([
            'name' => 'Website redesign',
            'rounding_minutes' => 15,
        ]);
        Project::factory()->for($acme)->capped(6000, 200000)->create([
            'name' => 'Support retainer',
            'rounding_minutes' => 6,
        ]);
        Project::factory()->for($acme)->hourly(5000)->archived()->create([
            'name' => 'Old intranet',
            'rounding_minutes' => 1,
        ]);

        $birch = Client::factory()->create([
            'name' => 'Birch & Daughters',
            'email' => 'accounts@birch.test',
            'vat_number' => null,
        ]);

        Project::factory()->for($birch)->fixed(150000)->create([
            'name' => 'Logo and brand book',
            'rounding_minutes' => 1,
        ]);
        Project::factory()->for($birch)->hourly(5500)->create([
            'name' => 'Webshop maintenance',
            'rounding_minutes' => 15,
        ]);

        $kala = Client::factory()->create([
            'name' => 'Kuressaare Kala AS',
            'email' => 'arved@kala.test',
            'vat_number' => 'EE100987654',
        ]);

        Project::factory()->for($kala)->fixed(480000)->create([
            'name' => 'Stock app MVP',
            'rounding_minutes' => 15,
        ]);
        Project::factory()->for($kala)->capped(7000, 350000)->create([
            'name' => 'Data migration',
            'rounding_minutes' => 6,
        ]);
    }
}
```

**What happens:**

1. `Client::factory()->create([...])` builds a client from the factory's defaults, overrides them with
   the array, and saves it (`INSERT`).
2. `Project::factory()->for($acme)` sets the `client` relationship, so `client_id` becomes Acme's id and
   the factory does **not** create an extra client.
3. `->hourly(6500)` applies a state; `->archived()` applies a second state on top.
4. The values match the table in [Concepts §8](README.md#8-seeding-and-factories): 3 clients, 7 projects,
   every billing type at least once, one archived project.

Rebuild the database from scratch and seed it:

```bash
php artisan migrate:fresh --seed
```

**What happens:** `migrate:fresh` **drops every table**, runs all migrations from the beginning, and then
`--seed` runs `DatabaseSeeder`. Use it only on your development database — never on a database with real
data.

**Check it works:**

```bash
php artisan tinker --execute="echo App\Models\Client::count().' clients, '.App\Models\Project::count().' projects';"
```

The output is `3 clients, 7 projects`. (You may also open the tables in your database tool — reading is
fine, changing is not.)

**Commit:** `feat(db): add factories and development seed data`

---

## Step 9 — First queries in Tinker

**Why:** Tinker is an interactive PHP shell with your whole application loaded. It is the fastest way to
try a query and see its SQL.

```bash
php artisan tinker
```

First import the classes, so you can write short names:

```php
use App\Models\Client;
use App\Models\Project;
use App\Enums\BillingType;
```

Now type these one by one. The comment shows what you should see (shortened).

```php
Client::count();
// = 3

Client::orderBy('name')->pluck('name');
// = Illuminate\Support\Collection { all: ["Acme OÜ", "Birch & Daughters", "Kuressaare Kala AS"] }

Client::find(1);
// = App\Models\Client { id: 1, name: "Acme OÜ", email: "billing@acme.test", ... }

Client::find(999);
// = null

$acme = Client::where('email', 'billing@acme.test')->first();
$acme->projects()->count();
// = 3

$acme->projects->pluck('name');
// = ["Website redesign", "Support retainer", "Old intranet"]

Project::where('billing_type', BillingType::Fixed)->orderBy('name')->pluck('name');
// = ["Logo and brand book", "Stock app MVP"]

$p = Project::first();
$p->billing_type;
// = App\Enums\BillingType { name: "Hourly", value: "hourly" }
$p->billing_type->label();
// = "Hourly"
$p->client->name;
// = "Acme OÜ"
```

**What happens:**

1. `Client::count()` runs `select count(*) as aggregate from "clients"` — the database counts, PHP gets
   one number.
2. `orderBy('name')->pluck('name')` runs `select "name" from "clients" order by "name" asc` and returns
   only the names.
3. `find(1)` looks up the primary key. It returns `null` if the row does not exist. (Its sibling
   `findOrFail(1)` throws an exception instead — lesson 04 uses that to show a 404 page.)
4. `where(...)->first()` adds `limit 1` and returns one model or `null`.
5. `where('billing_type', BillingType::Fixed)` — you pass the enum; Eloquent sends its value `fixed`.
6. `$p->client` follows the relationship: a second query, `select * from "clients" where "clients"."id" = ?`.

### Seeing the SQL

Without running the query:

```php
Project::where('billing_type', BillingType::Hourly)->orderBy('name')->toSql();
// = "select * from "projects" where "billing_type" = ? order by "name" asc"

Project::where('billing_type', BillingType::Hourly)->orderBy('name')->toRawSql();
// = "select * from "projects" where "billing_type" = 'hourly' order by "name" asc"
```

`toSql()` shows the SQL with `?` placeholders — exactly what is sent to PostgreSQL. The values travel
separately. `toRawSql()` fills the values in, for reading only; it is not what actually runs.

Recording every query that runs:

```php
DB::enableQueryLog();

$acme = Client::where('email', 'billing@acme.test')->first();
$acme->projects->count();
$acme->projects->count();

DB::getQueryLog();
// = [
//     ["query" => "select * from "clients" where "email" = ? limit 1", "bindings" => ["billing@acme.test"], "time" => 0.61],
//     ["query" => "select * from "projects" where "projects"."client_id" = ? and "projects"."client_id" is not null", "bindings" => [1], "time" => 0.42],
//   ]
```

**What happens:** there are **two** queries, not three. The first `$acme->projects` loaded the projects
and stored them on the `$acme` object; the second call used the stored collection. This caching is useful —
and it is also why the N+1 problem in lesson 06 is easy to create without noticing.

Loading a relation the moment you first touch it, like `$acme->projects` here, is called **lazy
loading**. Laravel can make it throw an exception in development: `Model::preventLazyLoading()`, the
second guard that `Model::shouldBeStrict()` switches on. We leave it off today, so you can first see the
N+1 problem with your own eyes in lesson 06; there `shouldBeStrict()` replaces the
`preventSilentlyDiscardingAttributes` line in `AppServiceProvider`.

Leave Tinker with `exit`.

**Check it works:** you got the same results as above. If your ids are different, you ran the seeder more
than once without `migrate:fresh` — run `php artisan migrate:fresh --seed` again.

---

## Step 10 — Real data on the home page

**Why:** a visible result, and our first controller that reads from the database.

Open `app/Http/Controllers/HomeController.php` from lesson 01. It has an `index()` method that passes
`today` to the view, and (from the lesson 01 independent work) an `about()` method. **Keep both.** We only
add two values to the array in `index()` and two imports at the top:

```php
<?php

// app/Http/Controllers/HomeController.php

namespace App\Http\Controllers;

use App\Models\Client;
use App\Models\Project;
use Illuminate\View\View;

class HomeController extends Controller
{
    public function index(): View
    {
        return view('home', [
            'today' => today(),
            'clientCount' => Client::count(),
            'projectCount' => Project::count(),
        ]);
    }

    public function about(): View
    {
        return view('about');
    }
}
```

In `resources/views/home.blade.php`, add this inside the `@section('content')` block, under the
"Today is …" paragraph:

```blade
{{-- resources/views/home.blade.php (inside @section('content'), under "Today is …") --}}
<p>
    You have <strong>{{ $clientCount }}</strong> clients
    and <strong>{{ $projectCount }}</strong> projects.
</p>
```

**What happens:**

1. The controller runs two `count(*)` queries and passes two integers to the view, next to `today`.
2. The view only prints them. It does not query the database itself — views display, controllers fetch.
   Lesson 04 explains why.
3. Lesson 04–06 controllers talk to models directly; lesson 07 moves this into services.

**Check it works:** start the server (`php artisan serve`, or use Herd) and open the home page. You see
"You have **3** clients and **7** projects."

Run the formatter check the same way CI does:

```bash
./vendor/bin/pint --test
```

**Commit:** `feat(home): show client and project counts`

---

## Troubleshooting

| Error / symptom | Cause | Fix |
|---|---|---|
| `SQLSTATE[08006] [7] connection to server at "127.0.0.1", port 5432 failed: Connection refused` | PostgreSQL container is not running | Start Docker Desktop, run `docker compose up -d` |
| `SQLSTATE[08006] ... password authentication failed for user "billable"` | `.env` does not match `compose.yaml` | Check `DB_USERNAME`, `DB_PASSWORD`, `DB_DATABASE`; then `php artisan config:clear` |
| `SQLSTATE[42P01]: Undefined table: 7 ERROR: relation "clients" does not exist` | You forgot `php artisan migrate` | Run it |
| `SQLSTATE[42P01] ... relation "clients" does not exist` **while migrating** `projects` | The projects migration's timestamp is older than the clients one | Rename the projects migration file so its timestamp is later (it has not run yet, so this is allowed) |
| `SQLSTATE[23503]: Foreign key violation ... update or delete on table "clients" violates foreign key constraint "projects_client_id_foreign"` | You tried to delete a client that has projects | That is `restrictOnDelete()` working. Delete the projects first, or keep the client |
| `SQLSTATE[23505]: Unique violation ... clients_email_unique` | Running `db:seed` twice inserts the same emails again | Use `php artisan migrate:fresh --seed` |
| `Add fillable property [archived_at] to allow mass assignment on [App\Models\Project].` | Mass assignment of a field not in `$fillable` (only reported because of `preventSilentlyDiscardingAttributes`, step 6) | This is the protection working. Set the field explicitly: `$project->archived_at = now(); $project->save();` |
| A field you passed to `create()` / `update()` is empty in the database, no error | The field is not in `$fillable` and the guard from step 6 is missing, so Eloquent dropped it silently | Add `Model::preventSilentlyDiscardingAttributes(...)` to `AppServiceProvider::boot()`; then add the field to `$fillable` if users may set it |
| `SQLSTATE[23502]: Not null violation ... null value in column "created_at"` | A row inserted past Eloquent (`DB::table(...)->insert()`, raw SQL) without the two dates | Create rows through the model or a factory; with the query builder, pass `'created_at' => now(), 'updated_at' => now()` |
| `ValueError: "weekly" is not a valid backing value for enum App\Enums\BillingType` | A row or input contains a value that is not an enum case | Fix the data or input; never add a string that is not a case |
| Tinker: `Class "Client" not found` | You did not import the class in this Tinker session | `use App\Models\Client;` or write `App\Models\Client::count()` |
| `Call to undefined method App\Models\Client::factory()` | `use HasFactory;` missing in the model | Add the trait (step 5) |
| You changed a migration and `migrate` says `Nothing to migrate` | Laravel already ran that file | Local and never pushed: `php artisan migrate:rollback`, edit, migrate. Otherwise: a new migration |

---

## Recap

- `app/Enums/BillingType.php` — backed string enum with `label()`.
- Two migrations in `database/migrations/` create `clients` and `projects` with the exact columns,
  a unique email, a restricting foreign key, an index on `client_id`, and `NOT NULL`
  `created_at` / `updated_at` columns.
- `app/Models/Client.php` and `app/Models/Project.php` — Active Record models with `$fillable`, casts
  and the `hasMany` / `belongsTo` relationship.
- `app/Providers/AppServiceProvider.php` — `Model::preventSilentlyDiscardingAttributes()` outside
  production.
- `database/factories/ClientFactory.php`, `ProjectFactory.php` (with billing-type states), and
  `database/seeders/DatabaseSeeder.php` with fixed development data.
- `php artisan migrate:fresh --seed` rebuilds everything from nothing.
- `toSql()`, `toRawSql()` and `DB::enableQueryLog()` show which SQL runs.
- `HomeController@index` and `home.blade.php` show live counts.

---

## Independent work — solutions

> **The tasks are not here.** The task descriptions (Basic / Intermediate / Advanced, with acceptance
> criteria) are in this lesson's [README.md → Independent work (~2 h)](README.md#independent-work-2-h).
> Read them there first and build the features in your project. This section only contains the
> solutions — try each task yourself, then open a solution to compare or when you are stuck.

<details>
<summary><strong>Basic 1</strong> — Practice queries</summary>

Run `php artisan migrate:fresh --seed` first, then `php artisan tinker` and import the classes
(`use App\Models\Client; use App\Models\Project; use App\Enums\BillingType;`).

```php
// Q1 — all client names, A–Z
Client::orderBy('name')->pluck('name');
// select "name" from "clients" order by "name" asc

// Q2 — number of projects
Project::count();
// select count(*) as aggregate from "projects"

// Q3 — hourly projects, A–Z
Project::where('billing_type', BillingType::Hourly)->orderBy('name')->pluck('name');
// select "name" from "projects" where "billing_type" = ? order by "name" asc

// Q4 — hourly rate €60.00 or more
Project::where('hourly_rate_cents', '>=', 6000)->orderBy('name')->get(['name', 'hourly_rate_cents']);
// select "name", "hourly_rate_cents" from "projects" where "hourly_rate_cents" >= ? order by "name" asc

// Q5 — clients without a VAT number
Client::whereNull('vat_number')->pluck('name');
// select "name" from "clients" where "vat_number" is null

// Q6 — archived projects
Project::whereNotNull('archived_at')->pluck('name');
// select "name" from "projects" where "archived_at" is not null

// Q7 — how many projects does Acme have?
Client::where('email', 'billing@acme.test')->firstOrFail()->projects()->count();
// select * from "clients" where "email" = ? limit 1
// select count(*) as aggregate from "projects" where "projects"."client_id" = ? and "projects"."client_id" is not null

// Q8 — most expensive fixed-price project
Project::where('billing_type', BillingType::Fixed)->orderByDesc('fixed_price_cents')->first(['name', 'fixed_price_cents']);
// select "name", "fixed_price_cents" from "projects" where "billing_type" = ? order by "fixed_price_cents" desc limit 1
```

Notes:

- Q4: projects with `NULL` hourly rate (fixed ones) are not included — in SQL, `NULL >= 6000` is not true.
- Q7: `->projects()->count()` counts in the database. `->projects->count()` would load all projects and
  count in PHP — it gives the same answer here, but the acceptance criteria ask you not to do that.
- Q8: `orderByDesc(...)->first()` is better than `->get()->sortByDesc(...)->first()`, which loads all rows.
- To get the SQL of each line, put `->toSql()` in place of the last call, or wrap the line in
  `DB::enableQueryLog()` … `DB::getQueryLog()`.

</details>

<details>
<summary><strong>Basic 2</strong> — <code>Project</code> query methods (local scopes)</summary>

A **local scope** is a method on the model that adds conditions to a query. You call it without the
`scope` prefix, and you can chain it.

Add to `app/Models/Project.php` (and add `use Illuminate\Database\Eloquent\Builder;` to the imports):

```php
// app/Models/Project.php — add inside the class, after isArchived()

    /**
     * Only projects that are not archived.
     *
     * @param  Builder<Project>  $query
     */
    public function scopeActive(Builder $query): void
    {
        $query->whereNull('archived_at');
    }

    /**
     * Only projects with the given billing type.
     *
     * @param  Builder<Project>  $query
     */
    public function scopeOfBillingType(Builder $query, BillingType $type): void
    {
        $query->where('billing_type', $type);
    }
```

Usage:

```php
$acme = Client::where('email', 'billing@acme.test')->first();

$acme->projects()->active()->orderBy('name')->pluck('name');
// = ["Support retainer", "Website redesign"]
// select "name" from "projects" where "projects"."client_id" = ? and "projects"."client_id" is not null
//   and "archived_at" is null order by "name" asc

Project::ofBillingType(BillingType::Capped)->orderBy('name')->pluck('name');
// = ["Data migration", "Support retainer"]
```

Why the parameter type is `BillingType`, not `string`: `ofBillingType('caped')` would be a silent bug
that returns nothing. With the enum type, PHP rejects the call.

Newer Laravel versions also offer a `#[Scope]` attribute on a `protected` method without the `scope`
prefix. The `scopeXxx` style above works in every version and is what you will see in most existing code.

</details>

<details>
<summary><strong>Intermediate</strong> — Check constraint on <code>rounding_minutes</code></summary>

```bash
php artisan make:migration add_rounding_minutes_check_to_projects_table
```

```php
<?php

// database/migrations/YYYY_MM_DD_HHMMSS_add_rounding_minutes_check_to_projects_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Support\Facades\DB;

return new class extends Migration
{
    public function up(): void
    {
        // SQLite (used by some test setups) cannot add a constraint to an existing table.
        if (DB::getDriverName() !== 'pgsql') {
            return;
        }

        DB::statement(
            'ALTER TABLE projects ADD CONSTRAINT projects_rounding_minutes_check CHECK (rounding_minutes IN (1, 6, 15))'
        );
    }

    public function down(): void
    {
        if (DB::getDriverName() !== 'pgsql') {
            return;
        }

        DB::statement('ALTER TABLE projects DROP CONSTRAINT projects_rounding_minutes_check');
    }
};
```

Why `DB::statement`: Laravel's schema builder has no method for check constraints, so we write the SQL.
It is still a migration — versioned, in Git, with a `down()`.

Why the driver check: if you later run feature tests on SQLite in memory (lesson 14 discusses this), the
`ALTER TABLE ... ADD CONSTRAINT` statement would fail there. The rule is still enforced in PostgreSQL,
which is the real database.

Test it:

```bash
php artisan migrate
php artisan tinker
```

```php
$p = App\Models\Project::first();
$p->rounding_minutes = 5;
$p->save();
// Illuminate\Database\QueryException  SQLSTATE[23514]: Check violation: 7 ERROR:  new row for relation
// "projects" violates check constraint "projects_rounding_minutes_check"
```

Then check that `php artisan migrate:fresh --seed` still works (all seed values are 1, 6 or 15) and that
`php artisan migrate:rollback` removes the constraint.

</details>

<details>
<summary><strong>Advanced</strong> — When to use raw SQL (example answer)</summary>

Situations where raw SQL is the better tool:

1. **Reports and aggregates** with several `GROUP BY` levels, `HAVING`, window functions
   (`ROW_NUMBER() OVER ...`) or `generate_series` for "every week, including empty ones". The ORM can
   express some of it, but the SQL is often shorter and easier to check.
2. **Bulk operations** on many rows: `UPDATE projects SET archived_at = now() WHERE ...` runs one
   statement instead of loading thousands of models and saving each one.
3. **Database features the ORM does not know**: check constraints, partial indexes, `SELECT ... FOR UPDATE
   SKIP LOCKED`, full-text search.
4. **Performance**: when you have measured that the ORM's query is slow and a hand-written one is faster.

Example — new projects per week for the last 8 weeks, **including weeks with none**:

```php
use Illuminate\Support\Carbon;
use Illuminate\Support\Facades\DB;

$to = now()->startOfWeek(Carbon::MONDAY); // Monday 00:00 of this week
$from = $to->copy()->subWeeks(7); // 8 weeks in total

$rows = DB::select(<<<'SQL'
    SELECT CAST(w.week_start AS date) AS week_start,
           COUNT(p.id)                AS new_projects
    FROM generate_series(CAST(? AS timestamp), CAST(? AS timestamp), interval '1 week')
         AS w(week_start)
    LEFT JOIN projects p
           ON p.created_at >= w.week_start
          AND p.created_at <  w.week_start + interval '1 week'
    GROUP BY w.week_start
    ORDER BY w.week_start
    SQL, [$from, $to]);
```

- `generate_series` makes one row per Monday, even when no project was created that week. The
  `LEFT JOIN` keeps those rows, and `COUNT(p.id)` gives `0` for them. Eloquent has no method for this;
  you would have to fill the gaps in PHP.
- The two dates go in the bindings array, **never** into the SQL string. PostgreSQL receives the SQL with
  `?` and the values separately. A value like `'; DROP TABLE clients; --` is only ever treated as a
  value — here PostgreSQL rejects it as an invalid timestamp, and nothing is dropped.
- `CAST(? AS timestamp)` tells PostgreSQL the type of the parameter; without it, it cannot choose
  between the `timestamp` and `timestamptz` versions of `generate_series`.
- `DB::select` returns plain `stdClass` objects, not models: no casts, no relationships.

Expected result if you seeded this week: 8 rows; the last one (this week) has `new_projects = 7`, the
others `0`.

Downsides: the SQL is tied to PostgreSQL (`generate_series`); you lose casts and relationships; a renamed
column is not found by your editor's refactoring tools. Use raw SQL where it clearly helps, keep it in
one place (later: a repository or a report class), and cover it with a test.

**When *not* to use raw SQL:** "number of projects and total budget cap per client" looks like a report,
but the ORM does it well: `Client::withCount('projects')->withSum('projects', 'budget_cap_cents')->get()`.
For queries in between, the query builder — `DB::table('clients')->leftJoin(...)->groupBy(...)
->selectRaw(...)` — is a useful middle way.

</details>
