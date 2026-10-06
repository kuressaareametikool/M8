# 05 — MVC: forms and validation · Laravel

[← Concepts](README.md) · Starting point: end of lesson 04 · Estimated time in class: 2 h

---

## What we build today

- Full resource routes for clients: `create`, `store`, `edit`, `update`, `destroy`.
- Two Form Requests: `StoreClientRequest` and `UpdateClientRequest` (unique e-mail that ignores the
  client being edited).
- One form partial `clients/_form.blade.php`, used by `create.blade.php` and `edit.blade.php`.
- Flash messages (`status` = green, `error` = red) shown by the layout.
- Delete with a confirm dialog, and a friendly message when the client still has projects.

**Final result:** on `/clients` there is a "New client" button. On a client page there are "Edit" and
"Delete" buttons. Wrong input comes back with your text still in the fields and a red message under
each wrong field. Every successful action ends on a page with a green message.

This code builds on the lesson 03–04 state: `App\Models\Client` with
`protected $fillable = ['name', 'email', 'vat_number'];` and a `projects()` `HasMany` relation,
`ClientController` with `index()` and `show()`, the layout `resources/views/layouts/app.blade.php`
(with `@yield('title')`, `@yield('content')` and `@include('partials.flash')`), and the seed data
(3 clients, 7 projects; `php artisan migrate:fresh --seed` restores it).

---

## Step 1 — Open up the resource routes

In lesson 04 you registered only two of the seven resource routes. Now we need all of them.

Open `routes/web.php` and find this line from lesson 04:

```php
// routes/web.php (lesson 04 — old line)
Route::resource('clients', ClientController::class)->only(['index', 'show']);
```

Replace it with:

```php
// routes/web.php
Route::resource('clients', ClientController::class);
```

Check the result:

```bash
php artisan route:list --path=clients
```

**What happens:**

1. `Route::resource` registers seven routes with conventional names and methods:

   | Method | URL | Controller method | Route name |
   |---|---|---|---|
   | GET | `/clients` | `index` | `clients.index` |
   | GET | `/clients/create` | `create` | `clients.create` |
   | POST | `/clients` | `store` | `clients.store` |
   | GET | `/clients/{client}` | `show` | `clients.show` |
   | GET | `/clients/{client}/edit` | `edit` | `clients.edit` |
   | PUT/PATCH | `/clients/{client}` | `update` | `clients.update` |
   | DELETE | `/clients/{client}` | `destroy` | `clients.destroy` |

2. `/clients/create` is registered **before** `/clients/{client}`. Otherwise Laravel would try to find a
   client with the id `create` and answer 404. The resource method handles the order for you.
3. `{client}` uses **route model binding** (lesson 04): Laravel loads the `Client` with that id or
   answers 404.

**Check it works:** the `route:list` output shows 7 lines for `clients` (PUT and PATCH share one line
as `PUT|PATCH`).

---

## Step 2 — Flash messages: success and error

Controllers will put a message into the session just before they redirect. The layout shows it on the
next page. We use two keys: `status` for success and `error` for problems.

In lesson 04 you created `resources/views/partials/flash.blade.php` (the layout includes it with
`@include('partials.flash')`, directly above `@yield('content')`). So far it shows only `status`.
Replace the whole file with:

```blade
{{-- resources/views/partials/flash.blade.php --}}
@if (session('status'))
    <article class="flash flash-success" role="status">{{ session('status') }}</article>
@endif
@if (session('error'))
    <article class="flash flash-error" role="alert">{{ session('error') }}</article>
@endif
```

And add a few lines of CSS inside `<head>`, after the Pico.css `<link>`:

```blade
{{-- resources/views/layouts/app.blade.php (inside <head>, after the Pico link) --}}
<style>
    .flash { padding: 0.75rem 1rem; }
    .flash-success { border-left: 0.35rem solid #2e7d32; }
    .flash-error { border-left: 0.35rem solid #c62828; }
</style>
```

**What happens:**

1. `session('status')` reads a value from the session. Values stored with `->with('status', ...)` on a
   redirect are **flash data**: Laravel keeps them for exactly one more request and then deletes them.
2. `role="status"` and `role="alert"` tell screen readers to announce the message.
3. `{{ }}` escapes HTML, so a client name with `<script>` in it is shown as text, never run.

**Check it works:** nothing visible yet. We test it in step 4. Because every page gets the partial
through the layout, controllers only have to set the message — no page template needs to change.

---

## Step 3 — The Form Request for creating a client

A **Form Request** is a class that holds the validation rules for one form. Laravel runs it
automatically **before** the controller method. If the rules fail, the controller method never runs.

```bash
php artisan make:request StoreClientRequest
```

Replace the generated file with:

```php
<?php

// app/Http/Requests/StoreClientRequest.php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class StoreClientRequest extends FormRequest
{
    /**
     * Billable has no login (single user), so everybody may send this form.
     */
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
            'name' => ['required', 'string', 'max:120'],
            'email' => ['required', 'string', 'email', 'max:255', Rule::unique('clients', 'email')],
            'vat_number' => ['nullable', 'string', 'regex:/^EE\d{9}$/'],
        ];
    }

    /**
     * Human names for fields, used inside the default messages.
     *
     * @return array<string, string>
     */
    public function attributes(): array
    {
        return [
            'email' => 'e-mail',
            'vat_number' => 'VAT number',
        ];
    }

    /**
     * @return array<string, string>
     */
    public function messages(): array
    {
        return [
            'vat_number.regex' => 'The VAT number must be EE followed by 9 digits, for example EE101234567.',
        ];
    }

    /**
     * Normalise the input before the rules run.
     */
    protected function prepareForValidation(): void
    {
        $email = $this->input('email');
        $vatNumber = $this->input('vat_number');

        $this->merge([
            'email' => is_string($email) ? mb_strtolower($email) : $email,
            'vat_number' => is_string($vatNumber) ? strtoupper(str_replace(' ', '', $vatNumber)) : $vatNumber,
        ]);
    }
}
```

**What happens:**

1. `authorize()` must return `true`, otherwise Laravel answers `403 This action is unauthorized.` We have
   no users, so every request is allowed.
2. `rules()` returns one array of rules per field. We use **arrays** instead of the pipe string
   `'required|max:120'`, because a regex can contain `|` and would be split in the wrong place.
3. `Rule::unique('clients', 'email')` runs `SELECT count(*) FROM clients WHERE email = ?`. If the
   count is not 0, the field fails with "The e-mail has already been taken."
4. `nullable` means "empty is fine, and then skip the other rules". It works because Laravel has two
   global middleware: `TrimStrings` (removes spaces at both ends of every input) and
   `ConvertEmptyStringsToNull` (turns `""` into `null`). An empty VAT field arrives as `null`.
5. `prepareForValidation()` runs **before** the rules. We lower-case the e-mail and remove spaces from
   the VAT number and upper-case it. So `ee 101 234 567` becomes `EE101234567` and passes. PostgreSQL
   compares strings case-sensitively, so lower-casing e-mails keeps the unique rule honest.
6. `attributes()` changes `vat_number` to `VAT number` in messages like "The VAT number field must be a
   string." `messages()` replaces one specific message (`field.rule`) with our own text.

---

## Step 4 — `create` and `store`

Open `app/Http/Controllers/ClientController.php`. Keep `index()` and `show()` from lesson 04 and add the
new methods. The complete controller after today (`index()` and `show()` are the lesson 04 versions; if
you did the search/sort homework, keep your `index()`):

```php
<?php

// app/Http/Controllers/ClientController.php

namespace App\Http\Controllers;

use App\Http\Requests\StoreClientRequest;
use App\Http\Requests\UpdateClientRequest;
use App\Models\Client;
use Illuminate\Http\RedirectResponse;
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

    public function create(): View
    {
        return view('clients.create', ['client' => new Client]);
    }

    public function store(StoreClientRequest $request): RedirectResponse
    {
        $client = Client::create($request->validated());

        return redirect()
            ->route('clients.show', $client)
            ->with('status', "Client {$client->name} created.");
    }

    public function edit(Client $client): View
    {
        return view('clients.edit', ['client' => $client]);
    }

    public function update(UpdateClientRequest $request, Client $client): RedirectResponse
    {
        $client->update($request->validated());

        return redirect()
            ->route('clients.show', $client)
            ->with('status', 'Client updated.');
    }

    public function destroy(Client $client): RedirectResponse
    {
        $projectCount = $client->projects()->count();

        if ($projectCount > 0) {
            return redirect()
                ->route('clients.show', $client)
                ->with('error', "{$client->name} still has {$projectCount} project(s). Delete or archive them first.");
        }

        $client->delete();

        return redirect()
            ->route('clients.index')
            ->with('status', "Client {$client->name} deleted.");
    }
}
```

`UpdateClientRequest` does not exist yet — we create it in step 6. Until then, comment out the `use`
line and the `update()` method, or do step 6 first if you prefer.

**What happens in `create()`:** we pass an **empty** `Client` (`new Client`, not saved). The form partial
can then always write `$client->name`: for a new client it is `null`, for an existing one it is the
stored value. One partial, no `if`.

**What happens in `store()`** — follow one submit:

1. The browser sends `POST /clients` with `_token`, `name`, `email`, `vat_number`.
2. Middleware `ValidateCsrfToken` compares `_token` with the token in the session. Wrong or missing →
   `419 Page Expired`, and nothing else runs.
3. Laravel sees the type-hint `StoreClientRequest`, creates it, runs `prepareForValidation()`, then
   `rules()`.
4. **If validation fails:** Laravel stores the errors and the old input in the session and redirects
   **back** to the form (`302`, `Location: /clients/create`). `store()` never runs.
5. **If validation passes:** `store()` runs. `$request->validated()` returns **only** the three fields
   that have rules — an extra `id` or `is_vip` field in the request is dropped.
6. `Client::create()` checks the array against `$fillable` (second layer of mass assignment
   protection) and runs `INSERT`.
7. `redirect()->route('clients.show', $client)` answers `302` with `Location: /clients/7`.
   `->with('status', ...)` flashes the message. This is Post/Redirect/Get.
8. The browser requests `GET /clients/7`, the layout finds `session('status')` and shows the green box.

Now the views. First the **partial**, the fields shared by create and edit:

```blade
{{-- resources/views/clients/_form.blade.php --}}
@csrf

<label for="name">
    Name
    <input id="name" name="name" type="text" maxlength="120" required
           value="{{ old('name', $client->name) }}"
           @error('name') aria-invalid="true" aria-describedby="name-error" @enderror>
    @error('name')
        <small id="name-error">{{ $message }}</small>
    @enderror
</label>

<label for="email">
    E-mail
    <input id="email" name="email" type="email" maxlength="255" required
           value="{{ old('email', $client->email) }}"
           @error('email') aria-invalid="true" aria-describedby="email-error" @enderror>
    @error('email')
        <small id="email-error">{{ $message }}</small>
    @enderror
</label>

<label for="vat_number">
    VAT number <small>(optional, e.g. EE101234567)</small>
    <input id="vat_number" name="vat_number" type="text" maxlength="20"
           value="{{ old('vat_number', $client->vat_number) }}"
           @error('vat_number') aria-invalid="true" aria-describedby="vat_number-error" @enderror>
    @error('vat_number')
        <small id="vat_number-error">{{ $message }}</small>
    @enderror
</label>

<button type="submit">{{ $submitLabel }}</button>
```

Then the create page:

```blade
{{-- resources/views/clients/create.blade.php --}}
@extends('layouts.app')

@section('title', 'New client')

@section('content')
    <h1>New client</h1>

    <form method="POST" action="{{ route('clients.store') }}">
        @include('clients._form', ['submitLabel' => 'Create client'])
    </form>

    <p><a href="{{ route('clients.index') }}">Back to clients</a></p>
@endsection
```

**What happens in the views:**

1. `@csrf` prints `<input type="hidden" name="_token" value="...">` with the session's token.
2. `old('name', $client->name)` returns the value the user sent last time **if** the previous request
   failed validation; otherwise it returns the second argument. On the create page that is `null` (empty
   field), on the edit page the stored name.
3. `@error('name') ... @enderror` renders its content only when the field `name` has an error. Inside it,
   `$message` is the first error message for that field.
4. `aria-invalid="true"` makes Pico.css draw a red border. `aria-describedby` links the input to the
   message, so a screen reader reads both.
5. `@include('clients._form', [...])` inserts the partial and passes an extra variable `submitLabel`.
   The partial also sees every variable of the parent view (here `$client`).
6. The underscore in `_form` is only a naming habit: "this is a piece, not a full page".

**Check it works:**

1. Open `http://localhost:8000/clients/create` (or `http://billable.test/clients/create` with Herd).
2. Fill in `Saar OÜ`, ` INFO@saar.ee `, `ee 101 234 567` and submit. You land on the new client's
   page with the green message "Client Saar OÜ created." The e-mail is stored as `info@saar.ee`, the VAT
   number as `EE101234567`.
3. Press F5. The page reloads, **no** "Resend form data?" dialog, and the green message is gone.
4. **Test the server-side rules.** The browser checks `required` and `type="email"` itself and would not
   even send the form. To see the server's answer, add `novalidate` to the `<form>` tag in
   `create.blade.php` for a moment:
   `<form method="POST" action="{{ route('clients.store') }}" novalidate>`.
   Submit an empty form: messages appear under Name and E-mail. Type the same e-mail as step 2: "The
   e-mail has already been taken." Type `EE123` as VAT: our own message. Your text stays in the fields.
5. Remove `novalidate` again.

**Commit:** `feat(clients): create clients with validation`

---

## Step 5 — See CSRF protection working

A two-minute experiment. In `_form.blade.php`, comment out `@csrf` (`{{-- @csrf --}}`), reload the
create page and submit a valid form.

**What you see:** `419 | Page Expired`. The request arrived without a token, and Laravel refused it
before any of our code ran. Put `@csrf` back.

**What happens:** Laravel's `web` middleware group includes `ValidateCsrfToken` (called
`VerifyCsrfToken` in older versions — tutorials and AI tools may show that name). It runs for every
POST, PUT, PATCH and DELETE request. A forged form on another website cannot know the token, so it gets
the same 419.

---

## Step 6 — `edit` and `update` with "unique, but ignore myself"

The update rules are almost the same as the store rules. Only the unique rule changes: it must ignore
the client being edited. So `UpdateClientRequest` **extends** `StoreClientRequest` and changes one rule.

```bash
php artisan make:request UpdateClientRequest
```

```php
<?php

// app/Http/Requests/UpdateClientRequest.php

namespace App\Http\Requests;

use Illuminate\Validation\Rule;

class UpdateClientRequest extends StoreClientRequest
{
    /**
     * @return array<string, array<int, mixed>>
     */
    public function rules(): array
    {
        $rules = parent::rules();

        $rules['email'] = [
            'required',
            'string',
            'email',
            'max:255',
            Rule::unique('clients', 'email')->ignore($this->route('client')),
        ];

        return $rules;
    }
}
```

**What happens:**

1. `extends StoreClientRequest` inherits `authorize()`, `attributes()`, `messages()` and
   `prepareForValidation()`. We only override `rules()`.
2. `$this->route('client')` returns the route parameter `{client}`. Because of route model binding it
   is already the `Client` model, not just the id.
3. `->ignore($client)` adds `AND id <> 7` to the unique query. The client's own e-mail no longer counts
   as a duplicate, but another client's e-mail still does.

The edit page. Note the one new line, `@method('PUT')`:

```blade
{{-- resources/views/clients/edit.blade.php --}}
@extends('layouts.app')

@section('title', 'Edit '.$client->name)

@section('content')
    <h1>Edit {{ $client->name }}</h1>

    <form method="POST" action="{{ route('clients.update', $client) }}">
        @method('PUT')
        @include('clients._form', ['submitLabel' => 'Save changes'])
    </form>

    <p><a href="{{ route('clients.show', $client) }}">Cancel</a></p>
@endsection
```

**What happens:**

1. HTML forms can only send GET or POST. `@method('PUT')` prints
   `<input type="hidden" name="_method" value="PUT">`.
2. Laravel sees `_method` on a POST request and treats it as PUT. This is **method spoofing**. The route
   `PUT /clients/{client}` then matches and calls `update()`.
3. `update()` receives the validated data and calls `$client->update(...)`, which fills the allowed
   fields and runs `UPDATE clients SET ... WHERE id = 7`. If nothing changed, Eloquent sends no query at
   all.
4. If validation fails, Laravel redirects back to `/clients/7/edit`, and `old()` refills the fields with
   what the user typed (not the stored values).

Now un-comment `update()` and the `use` line in the controller if you commented them out in step 4.

**Check it works:**

1. Open a client page, go to `/clients/{id}/edit` by hand (the button comes in step 8). The fields are
   filled with the stored values.
2. Change only the name and save. It works — the unchanged e-mail is not reported as "already taken".
3. Change the e-mail to the e-mail of **another** client (open `/clients` to find one). You get "The
   e-mail has already been taken." and the form shows what you typed.

**Commit:** `feat(clients): edit clients, unique e-mail ignores self`

---

## Step 7 — Delete with confirmation and a friendly FK error

Look at `destroy()` in the controller from step 4 again:

```php
// app/Http/Controllers/ClientController.php (destroy, already written in step 4)
public function destroy(Client $client): RedirectResponse
{
    $projectCount = $client->projects()->count();

    if ($projectCount > 0) {
        return redirect()
            ->route('clients.show', $client)
            ->with('error', "{$client->name} still has {$projectCount} project(s). Delete or archive them first.");
    }

    $client->delete();

    return redirect()
        ->route('clients.index')
        ->with('status', "Client {$client->name} deleted.");
}
```

**What happens:**

1. `$client->projects()` with parentheses returns the **query builder** for the relation, not the loaded
   collection. `->count()` runs `SELECT count(*) FROM projects WHERE client_id = ?` — one cheap query,
   no project objects are created.
2. If there are projects we do **not** try to delete. We redirect back to the client page with the red
   `error` flash. No exception, no 500 page.
3. Otherwise `$client->delete()` runs `DELETE FROM clients WHERE id = ?`. The `$client` object still
   exists in PHP memory after that, so we can use `$client->name` in the message.
4. The foreign key `projects.client_id` with restrict stays in the database as the safety net.

Without the check you would see this in the browser (with `APP_DEBUG=true`):

```
SQLSTATE[23503]: Foreign key violation: 7 ERROR: update or delete on table "clients"
violates foreign key constraint "projects_client_id_foreign" on table "projects"
```

`23503` is PostgreSQL's code for "foreign key violation". A real user would only see "500 Server Error".

---

## Step 8 — Buttons on the list and detail pages

On the clients list (`resources/views/clients/index.blade.php`), add a link below the `<h1>`:

```blade
{{-- resources/views/clients/index.blade.php (below the <h1>) --}}
<p><a href="{{ route('clients.create') }}" role="button">New client</a></p>
```

On the client page (`resources/views/clients/show.blade.php`), add this block below the client's
details and above the projects list:

```blade
{{-- resources/views/clients/show.blade.php (below the client details) --}}
<div class="grid">
    <a href="{{ route('clients.edit', $client) }}" role="button" class="secondary">Edit</a>

    <form method="POST" action="{{ route('clients.destroy', $client) }}"
          data-confirm="Delete client {{ $client->name }}? This cannot be undone."
          onsubmit="return confirm(this.dataset.confirm)">
        @csrf
        @method('DELETE')
        <button type="submit" class="contrast">Delete</button>
    </form>
</div>
```

**What happens:**

1. `role="button"` makes Pico.css draw a link as a button. It is still a normal GET link — fine,
   because showing a form changes nothing.
2. Delete is a **form**, not a link: it sends POST with `_method=DELETE`, so no crawler or link preview
   can trigger it, and CSRF protection applies.
3. `onsubmit="return confirm(...)"` shows the browser's OK/Cancel dialog. `return false` (Cancel) stops
   the submit.
4. The text lives in `data-confirm`, escaped by `{{ }}` like any attribute. `this.dataset.confirm`
   reads it back as plain text. A name like `O'Brien & Sons` is shown correctly and cannot break out
   into JavaScript.
5. `class="grid"` is Pico.css: the two children are shown side by side.

**Check it works:**

1. `/clients` shows "New client". A client page shows "Edit" and "Delete".
2. Create a new client (it has no projects), press Delete, press **Cancel**: nothing happens. Press
   Delete again, **OK**: you land on `/clients` with "Client … deleted."
3. Open a seeded client that has projects and press Delete → OK. You stay on the client page with the red
   message "… still has 2 project(s). Delete or archive them first." Nothing was deleted.

**Commit:** `feat(clients): delete clients with confirmation and project check`

---

## Step 9 — Format and final check

```bash
./vendor/bin/pint
php artisan route:list --path=clients
git status
```

**Check it works:** Pint reports the files it fixed (or nothing). Walk through the whole cycle once:
create → see green message → edit → change e-mail to a duplicate → see error → fix → save → delete.

**Commit:** `style: apply pint` (only if Pint changed something).

---

## Troubleshooting

| What you see | Cause | Fix |
|---|---|---|
| `419 Page Expired` | `@csrf` missing in the form, or the session expired (page open for hours) | Add `@csrf`; reload the form |
| `The POST method is not supported for route clients/7. Supported methods: GET, HEAD, PUT, PATCH, DELETE.` | `@method('PUT')` or `@method('DELETE')` missing | Add it inside the form |
| `Add [name] to fillable property to allow mass assignment on [App\Models\Client].` | `$fillable` on the model is missing or incomplete | `protected $fillable = ['name', 'email', 'vat_number'];` |
| `This action is unauthorized.` (403) | `authorize()` in the Form Request returns `false` (the generated default) | Return `true` |
| Errors never appear, form just reloads | Messages are not rendered in the view, or the field `name` attribute differs from the rule key | Check `name="vat_number"` matches `'vat_number' => [...]` and `@error('vat_number')` |
| `preg_match(): No ending delimiter '/' found` | Regex rule written in a pipe string: `'nullable\|regex:/^EE\d{9}$/'` breaks at `\|` | Use the array syntax |
| Editing without changing the e-mail says "already taken" | Using `StoreClientRequest` in `update()`, or `ignore()` missing | Type-hint `UpdateClientRequest` |
| `Route [clients.create] not defined.` | Routes still have `->only(['index', 'show'])` | Step 1 |
| `Undefined variable $client` in `_form` | `create()` does not pass `new Client` | `view('clients.create', ['client' => new Client])` |
| `SQLSTATE[23503]: Foreign key violation` | Deleting a client with projects without the check | The check in `destroy()` (step 7) |

---

## Recap

- `routes/web.php` — full `Route::resource('clients', ...)`.
- `app/Http/Requests/StoreClientRequest.php` — rules, messages, normalisation before validation.
- `app/Http/Requests/UpdateClientRequest.php` — extends the store request, unique rule ignores self.
- `app/Http/Controllers/ClientController.php` — `create`, `store`, `edit`, `update`, `destroy` with PRG
  and flash messages; project check before delete.
- `resources/views/clients/_form.blade.php` — one partial with `@csrf`, `old()`, `@error`.
- `resources/views/clients/create.blade.php`, `edit.blade.php` (with `@method('PUT')`).
- `resources/views/layouts/app.blade.php` — green and red flash boxes.
- `resources/views/clients/index.blade.php`, `show.blade.php` — New / Edit / Delete buttons.

---

## Independent work — solutions

<details>
<summary><strong>Basic — full CRUD for projects</strong></summary>

This assumes lesson 03–04: `App\Models\Project` with `client()` `BelongsTo`, `billing_type` cast to
`App\Enums\BillingType` (cases `Hourly`, `Fixed`, `Capped` with values `hourly`, `fixed`, `capped`
and a `label()` method), and `$fillable` containing `name`, `billing_type`, `hourly_rate_cents`,
`fixed_price_cents`, `budget_cap_cents`, `rounding_minutes` — but **not** `client_id` and not
`archived_at` (lesson 03). `client_id` comes from the URL through the relation, never from the form.

**1. Routes.** In lesson 04 you registered the project routes with `->shallow()->only(['index', 'show'])`.
Remove `->only(...)`:

```php
// routes/web.php
Route::resource('clients.projects', ProjectController::class)->shallow();
```

`shallow()` nests only the routes that need the parent (`index`, `create`, `store` under
`/clients/{client}/projects`) and keeps the others flat (`/projects/{project}`, `/projects/{project}/edit`).
The route names are `clients.projects.index`, `clients.projects.create`, `clients.projects.store`,
`projects.show`, `projects.edit`, `projects.update`, `projects.destroy` — the two names from lesson 04
stay the same. Run `php artisan route:list --path=projects` to see them.

**2. Money helpers.** Add these two functions to `app/helpers.php`, next to `euros()` from lesson 04:

```php
<?php

// app/helpers.php (add below euros())

if (! function_exists('euros_to_cents')) {
    /**
     * Converts validated user input like "85", "85.5", "85,50" to integer cents.
     * The input must already match /^\d{1,7}([.,]\d{1,2})?$/. No floats are used.
     */
    function euros_to_cents(string $euros): int
    {
        $normalized = str_replace(',', '.', trim($euros));
        [$whole, $fraction] = array_pad(explode('.', $normalized, 2), 2, '');

        return (int) $whole * 100 + (int) str_pad($fraction, 2, '0');
    }
}

if (! function_exists('cents_to_euros_input')) {
    /**
     * Converts cents to the text shown in a form field: 8550 → "85.50", null → "".
     */
    function cents_to_euros_input(?int $cents): string
    {
        if ($cents === null) {
            return '';
        }

        $sign = $cents < 0 ? '-' : '';
        $abs = abs($cents);

        return $sign.intdiv($abs, 100).'.'.str_pad((string) ($abs % 100), 2, '0', STR_PAD_LEFT);
    }
}
```

How `euros_to_cents('85,5')` works: `,` → `.` gives `85.5`; `explode` gives `['85', '5']`;
`str_pad('5', 2, '0')` pads on the right to `'50'`; `85 * 100 + 50 = 8550`. For `'85'` there is no dot,
`array_pad` adds an empty fraction `''`, padded to `'00'`. Only integers are multiplied.
`cents_to_euros_input()` works on the absolute value and puts the sign in front, so `-5` becomes `-0.05`
(without that, `-5 % 100` is `-5` and you would get `0.-5`).

We write the parsing by hand so you can see that no float is involved. PHP 8.4 also has a built-in exact
decimal type, `BcMath\Number` (needs the `bcmath` extension), and the `brick/money` package does all of
this for real projects.

**3. Form Requests.**

```php
<?php

// app/Http/Requests/StoreProjectRequest.php

namespace App\Http\Requests;

use App\Enums\BillingType;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class StoreProjectRequest extends FormRequest
{
    /** 1–7 digits, optionally followed by a dot or comma and 1–2 digits. */
    private const string EUROS = '/^\d{1,7}([.,]\d{1,2})?$/';

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
            'name' => ['required', 'string', 'max:120'],
            'billing_type' => ['required', Rule::enum(BillingType::class)],
            'hourly_rate' => ['exclude_unless:billing_type,hourly,capped', 'required', 'regex:'.self::EUROS],
            'fixed_price' => ['exclude_unless:billing_type,fixed', 'required', 'regex:'.self::EUROS],
            'budget_cap' => ['exclude_unless:billing_type,capped', 'required', 'regex:'.self::EUROS],
            'rounding_minutes' => ['required', 'integer', Rule::in([1, 6, 15])],
        ];
    }

    /**
     * @return array<string, string>
     */
    public function messages(): array
    {
        return [
            'hourly_rate.required' => 'Hourly and capped projects need an hourly rate.',
            'fixed_price.required' => 'Fixed price projects need a price.',
            'budget_cap.required' => 'Capped projects need a budget cap.',
            'hourly_rate.regex' => 'Use a format like 85 or 85.50.',
            'fixed_price.regex' => 'Use a format like 1200 or 1200.00.',
            'budget_cap.regex' => 'Use a format like 3000 or 3000.00.',
            'rounding_minutes.in' => 'Rounding must be 1, 6 or 15 minutes.',
        ];
    }

    /**
     * The validated data converted to database columns. Fields that do not
     * apply to the billing type were excluded by the rules and are stored as null.
     *
     * @return array<string, mixed>
     */
    public function projectData(): array
    {
        $data = $this->validated();

        return [
            'name' => $data['name'],
            'billing_type' => $this->enum('billing_type', BillingType::class),
            'hourly_rate_cents' => $this->cents($data, 'hourly_rate'),
            'fixed_price_cents' => $this->cents($data, 'fixed_price'),
            'budget_cap_cents' => $this->cents($data, 'budget_cap'),
            'rounding_minutes' => $this->integer('rounding_minutes'),
        ];
    }

    /**
     * @param  array<string, mixed>  $data
     */
    private function cents(array $data, string $field): ?int
    {
        return isset($data[$field]) ? euros_to_cents($data[$field]) : null;
    }
}
```

```php
<?php

// app/Http/Requests/UpdateProjectRequest.php

namespace App\Http\Requests;

class UpdateProjectRequest extends StoreProjectRequest
{
    // Same rules as creating. A separate class keeps the controller readable
    // and gives you a place for update-only rules later (for example R14).
}
```

Notes:

- `exclude_unless:billing_type,hourly,capped` means "ignore this field unless `billing_type` is `hourly`
  **or** `capped`". An ignored field is not validated and is **not** in `validated()`. The next rule,
  `required`, therefore only applies to the billing types that need the field. A leftover price in a
  hidden field of a fixed project causes no error and is not saved.
- `Rule::enum(BillingType::class)` accepts only the enum's backing values.
- `Rule::in([1, 6, 15])` is checked even though the form uses a `<select>`: a request can contain any
  value.
- `projectData()` keeps the euro → cents conversion out of the controller. A field missing from
  `validated()` becomes `null`, so switching a project from hourly to fixed clears the old rate.
  `$this->enum(...)` and `$this->integer(...)` read the input already converted to `BillingType` and `int`.
- `private const string` (a typed constant) needs PHP 8.3 or newer; we use PHP 8.4.

**4. Controller.**

```php
<?php

// app/Http/Controllers/ProjectController.php

namespace App\Http\Controllers;

use App\Enums\BillingType;
use App\Http\Requests\StoreProjectRequest;
use App\Http\Requests\UpdateProjectRequest;
use App\Models\Client;
use App\Models\Project;
use Illuminate\Http\RedirectResponse;
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

    public function create(Client $client): View
    {
        return view('projects.create', [
            'client' => $client,
            'project' => new Project(['billing_type' => BillingType::Hourly, 'rounding_minutes' => 15]),
            'billingTypes' => BillingType::cases(),
        ]);
    }

    public function store(StoreProjectRequest $request, Client $client): RedirectResponse
    {
        $project = $client->projects()->create($request->projectData());

        return redirect()
            ->route('projects.show', $project)
            ->with('status', "Project {$project->name} created.");
    }

    public function edit(Project $project): View
    {
        return view('projects.edit', [
            'project' => $project,
            'billingTypes' => BillingType::cases(),
        ]);
    }

    public function update(UpdateProjectRequest $request, Project $project): RedirectResponse
    {
        $project->update($request->projectData());

        return redirect()
            ->route('projects.show', $project)
            ->with('status', 'Project updated.');
    }

    public function destroy(Project $project): RedirectResponse
    {
        $client = $project->client;
        $project->delete();

        return redirect()
            ->route('clients.show', $client)
            ->with('status', "Project {$project->name} deleted.");
    }
}
```

`$client->projects()->create(...)` sets `client_id` automatically from the relation — the user cannot
choose another client by editing the form. Keep your own `index()` and `show()` if they differ.

**5. Views.** The partial:

```blade
{{-- resources/views/projects/_form.blade.php --}}
@csrf

@php($selectedType = old('billing_type', $project->billing_type?->value))

<label for="name">
    Name
    <input id="name" name="name" type="text" maxlength="120" required
           value="{{ old('name', $project->name) }}"
           @error('name') aria-invalid="true" @enderror>
    @error('name') <small>{{ $message }}</small> @enderror
</label>

<label for="billing_type">
    Billing type
    <select id="billing_type" name="billing_type" required @error('billing_type') aria-invalid="true" @enderror>
        @foreach ($billingTypes as $type)
            <option value="{{ $type->value }}" @selected($selectedType === $type->value)>{{ $type->label() }}</option>
        @endforeach
    </select>
    @error('billing_type') <small>{{ $message }}</small> @enderror
</label>

<div class="grid">
    <label for="hourly_rate">
        Hourly rate (€) <small>hourly and capped</small>
        <input id="hourly_rate" name="hourly_rate" type="text" inputmode="decimal"
               value="{{ old('hourly_rate', cents_to_euros_input($project->hourly_rate_cents)) }}"
               @error('hourly_rate') aria-invalid="true" @enderror>
        @error('hourly_rate') <small>{{ $message }}</small> @enderror
    </label>

    <label for="fixed_price">
        Fixed price (€) <small>fixed only</small>
        <input id="fixed_price" name="fixed_price" type="text" inputmode="decimal"
               value="{{ old('fixed_price', cents_to_euros_input($project->fixed_price_cents)) }}"
               @error('fixed_price') aria-invalid="true" @enderror>
        @error('fixed_price') <small>{{ $message }}</small> @enderror
    </label>

    <label for="budget_cap">
        Budget cap (€) <small>capped only</small>
        <input id="budget_cap" name="budget_cap" type="text" inputmode="decimal"
               value="{{ old('budget_cap', cents_to_euros_input($project->budget_cap_cents)) }}"
               @error('budget_cap') aria-invalid="true" @enderror>
        @error('budget_cap') <small>{{ $message }}</small> @enderror
    </label>
</div>

<label for="rounding_minutes">
    Round each entry up to
    <select id="rounding_minutes" name="rounding_minutes" @error('rounding_minutes') aria-invalid="true" @enderror>
        @foreach ([1, 6, 15] as $minutes)
            <option value="{{ $minutes }}" @selected((int) old('rounding_minutes', $project->rounding_minutes) === $minutes)>{{ $minutes }} min</option>
        @endforeach
    </select>
    @error('rounding_minutes') <small>{{ $message }}</small> @enderror
</label>

<button type="submit">{{ $submitLabel }}</button>
```

`type="text" inputmode="decimal"` instead of `type="number"`: a number input rejects `85,50` in some
browsers and locales, and phones still show a number keyboard with `inputmode`. `@selected(...)` prints
`selected` when the condition is true.

```blade
{{-- resources/views/projects/create.blade.php --}}
@extends('layouts.app')

@section('title', 'New project')

@section('content')
    <h1>New project for {{ $client->name }}</h1>

    <form method="POST" action="{{ route('clients.projects.store', $client) }}">
        @include('projects._form', ['submitLabel' => 'Create project'])
    </form>

    <p><a href="{{ route('clients.show', $client) }}">Back to {{ $client->name }}</a></p>
@endsection
```

```blade
{{-- resources/views/projects/edit.blade.php --}}
@extends('layouts.app')

@section('title', 'Edit '.$project->name)

@section('content')
    <h1>Edit {{ $project->name }}</h1>

    <form method="POST" action="{{ route('projects.update', $project) }}">
        @method('PUT')
        @include('projects._form', ['submitLabel' => 'Save changes'])
    </form>

    <p><a href="{{ route('projects.show', $project) }}">Cancel</a></p>
@endsection
```

On the client page add `<a href="{{ route('clients.projects.create', $client) }}" role="button">New project</a>`
above the projects list. On the project page add, below the details:

```blade
{{-- resources/views/projects/show.blade.php (below the details) --}}
<div class="grid">
    <a href="{{ route('projects.edit', $project) }}" role="button" class="secondary">Edit</a>
    <form method="POST" action="{{ route('projects.destroy', $project) }}"
          data-confirm="Delete project {{ $project->name }}?"
          onsubmit="return confirm(this.dataset.confirm)">
        @csrf
        @method('DELETE')
        <button type="submit" class="contrast">Delete</button>
    </form>
</div>
```

**Check:** create a capped project without a cap → "Capped projects need a budget cap." Enter `85,5` as
the rate → the project page shows `85,50 €` (your `euros()` format), the database has `8550`. Change the
type to fixed with a price → `hourly_rate_cents` becomes `null` (check with `php artisan tinker`,
`App\Models\Project::latest('id')->first()`).

</details>

<details>
<summary><strong>Intermediate — archive instead of delete</strong></summary>

**Model methods.** `Project` already has `isArchived()` and the `archived_at` datetime cast from
lesson 03. Add two methods below `isArchived()`:

```php
// app/Models/Project.php (add inside the class, below isArchived())
public function archive(): void
{
    $this->archived_at = now();
    $this->save();
}

public function unarchive(): void
{
    $this->archived_at = null;
    $this->save();
}
```

`archived_at` is not in `$fillable`, and that is correct: it is set only by these methods, never from a
form.

**Routes** — replace the resource line from the basic task:

```php
// routes/web.php
Route::resource('clients.projects', ProjectController::class)->shallow()->except(['destroy']);
Route::patch('/projects/{project}/archive', [ProjectController::class, 'archive'])->name('projects.archive');
Route::patch('/projects/{project}/unarchive', [ProjectController::class, 'unarchive'])->name('projects.unarchive');
```

**Controller** — remove `destroy()` and add:

```php
// app/Http/Controllers/ProjectController.php (replace destroy() with these)
public function archive(Project $project): RedirectResponse
{
    $project->archive();

    return redirect()
        ->route('projects.show', $project)
        ->with('status', "Project {$project->name} archived.");
}

public function unarchive(Project $project): RedirectResponse
{
    $project->unarchive();

    return redirect()
        ->route('projects.show', $project)
        ->with('status', "Project {$project->name} is active again.");
}
```

Sort archived projects last in `index()` (and in `ClientController::show()` if the client page lists
projects):

```php
// app/Http/Controllers/ProjectController.php (index)
$projects = $client->projects()
    ->orderByRaw('archived_at IS NOT NULL')
    ->orderBy('name')
    ->get();
```

`archived_at IS NOT NULL` is `false` for active projects and `true` for archived ones; `false` sorts
first.

**View** — replace the Delete form on the project page:

```blade
{{-- resources/views/projects/show.blade.php (replace the delete form) --}}
@if ($project->isArchived())
    <p><mark>Archived on {{ $project->archived_at->format('d.m.Y') }}</mark></p>
    <form method="POST" action="{{ route('projects.unarchive', $project) }}">
        @csrf
        @method('PATCH')
        <button type="submit" class="secondary">Unarchive</button>
    </form>
@else
    <form method="POST" action="{{ route('projects.archive', $project) }}"
          data-confirm="Archive project {{ $project->name }}?"
          onsubmit="return confirm(this.dataset.confirm)">
        @csrf
        @method('PATCH')
        <button type="submit" class="contrast">Archive</button>
    </form>
@endif
```

In the list, add `@if ($project->isArchived()) <small>(archived)</small> @endif` next to the name.

</details>

<details>
<summary><strong>Advanced — reusable <code>EstonianVatNumber</code> rule</strong></summary>

```bash
php artisan make:rule EstonianVatNumber
```

```php
<?php

// app/Rules/EstonianVatNumber.php

namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;

/**
 * An Estonian VAT number (KMKR number): "EE" followed by exactly 9 digits.
 * Combine with 'nullable' when the field is optional.
 */
class EstonianVatNumber implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if (! is_string($value) || preg_match('/^EE\d{9}$/', $value) !== 1) {
            $fail('The :attribute must be EE followed by 9 digits, for example EE101234567.');
        }
    }
}
```

Use it in `StoreClientRequest::rules()` (the update request inherits it):

```php
// app/Http/Requests/StoreClientRequest.php (rules)
'vat_number' => ['nullable', 'string', new EstonianVatNumber],
```

Add `use App\Rules\EstonianVatNumber;` at the top and remove the `vat_number.regex` entry from
`messages()` — the message now lives in the rule. `:attribute` is replaced with "VAT number" from
`attributes()`.

Why a class: the rule has a **name** that explains its meaning, the message lives next to the check,
and when the format changes you change one file. Because it is plain PHP, you can unit-test it in lesson
11 without a database.

Older tutorials show the `Rule` interface with `passes()` and `message()` methods. That interface is
deprecated; `ValidationRule` with `validate()` is the current form.

</details>
