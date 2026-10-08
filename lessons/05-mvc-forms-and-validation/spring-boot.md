# 05 — MVC: forms and validation · Spring Boot

[← Concepts](README.md) · Starting point: end of lesson 04 · Estimated time in class: 2 h · Independent work tasks: [README](README.md#independent-work-2-h)

---

## What we build today

- `common.FormBindingAdvice`: trims every text input and turns empty strings into `null`.
- `client.ClientForm`: a form-backing class with Jakarta Validation constraints.
- `ClientController`: `create`, `store`, `edit`, `update`, `destroy`, with Post/Redirect/Get and flash
  messages.
- A unique e-mail check that ignores the client being edited
  (`existsByEmailIgnoreCaseAndIdNot`).
- One Thymeleaf form fragment `clients/form.html`, used by `create.html` and `edit.html`.
- Delete with a confirm dialog and a friendly message when the client still has projects.
- `common.SecurityConfig`: Spring Security with no login (`permitAll()`), but CSRF protection on, so
  every POST form carries a hidden `_csrf` token — the same protection the Laravel track has.

**Final result:** on `/clients` there is a "New client" button. On a client page there are "Edit" and
"Delete" buttons. Wrong input comes back with your text still in the fields and a red message under
each wrong field. Every successful action ends on a page with a green message.

**Our URLs.** HTML forms can only send GET and POST. Instead of faking PUT and DELETE with a hidden
field, we give every action its own POST URL (see concept 2 in the [README](README.md)):

| Method | URL | Controller method | What it does |
|---|---|---|---|
| GET | `/clients` | `index` | list (lesson 04) |
| GET | `/clients/new` | `create` | empty form |
| POST | `/clients` | `store` | save new client |
| GET | `/clients/{id}` | `show` | detail page (lesson 04) |
| GET | `/clients/{id}/edit` | `edit` | filled form |
| POST | `/clients/{id}` | `update` | save changes |
| POST | `/clients/{id}/delete` | `destroy` | delete |

This code builds on the lesson 03–04 state: the `client.Client` entity (constructor
`Client(String name, String email, String vatNumber)`), `ClientRepository`, `ProjectRepository` (with
`countByClientId` and `findByClientIdOrderByName`), `ClientController` with `index()` and `show()`,
`common.NotFoundException` (annotated with `@ResponseStatus(NOT_FOUND)`), and `templates/layout.html`
with the one `layout(title, content)` fragment (head, nav, flash box, footer) that every page uses.
Start the app with the `dev` profile (`./mvnw spring-boot:run -Dspring-boot.run.profiles=dev`) so `DevDataSeeder` has created the 3 clients
and 7 projects.

---

## Step 1 — Empty strings become `null`

A browser sends an empty text field as an empty string `""`, not as "nothing". That causes two problems:

- `@Pattern(regexp = "^EE\\d{9}$")` on the optional VAT number would **fail** for `""`, because `""` does
  not match. (Jakarta constraints treat only `null` as "no value".)
- We would store `""` in `vat_number` instead of `NULL`.

Spring has a ready-made property editor for this. We register it once for all controllers:

```java
// src/main/java/ee/ta25/billable/common/FormBindingAdvice.java
package ee.ta25.billable.common;

import org.springframework.beans.propertyeditors.StringTrimmerEditor;
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.InitBinder;

/** Binding rules that apply to every form in the application. */
@ControllerAdvice
public class FormBindingAdvice {

  /** Trims all text input and converts empty strings to null. */
  @InitBinder
  public void trimStrings(WebDataBinder binder) {
    binder.registerCustomEditor(String.class, new StringTrimmerEditor(true));
  }
}
```

**What happens:**

1. `@ControllerAdvice` marks a class whose `@InitBinder`, `@ModelAttribute` and `@ExceptionHandler`
   methods apply to **all** controllers.
2. Before Spring copies request parameters into a form object, it calls `@InitBinder` methods to
   configure the `WebDataBinder` (the object that does the copying).
3. `StringTrimmerEditor(true)` removes spaces at both ends of every `String`. The `true` means "and if
   the result is empty, use `null`". Laravel does the same with its `TrimStrings` and
   `ConvertEmptyStringsToNull` middleware.

**Check it works:** the application still starts (`./mvnw spring-boot:run`). We see the effect in step 5.

---

## Step 2 — Flash messages: success and error

Controllers will store a message just before they redirect. The layout shows it on the next page. We use
two names: `status` for success and `error` for problems.

Since lesson 04, `templates/layout.html` has a flash box that shows only `status`:
`<div class="container" th:if="${status != null and timestamp == null}">`. Replace that `<div>` with
one that can show both:

```html
<!-- src/main/resources/templates/layout.html (replace the flash box between </header> and <main>) -->
<div class="container" th:if="${timestamp == null}">
  <article class="flash flash-success" role="status" th:if="${status}" th:text="${status}">
    Saved.
  </article>
  <article class="flash flash-error" role="alert" th:if="${error}" th:text="${error}">
    Something went wrong.
  </article>
</div>
```

And add a few CSS lines inside the layout's `<head>`, after the Pico.css `<link>`:

```html
<!-- src/main/resources/templates/layout.html (inside <head>, after the Pico link) -->
<style>
  .flash { padding: 0.75rem 1rem; }
  .flash-success { border-left: 0.35rem solid #2e7d32; }
  .flash-error { border-left: 0.35rem solid #c62828; }
</style>
```

No page needs a change: every page uses the layout, so every page shows the messages.

**What happens:**

1. A **flash attribute** is stored in the HTTP session by `RedirectAttributes.addFlashAttribute(...)`.
   On the **next** request, Spring moves it into the model and deletes it from the session. So it is
   shown exactly once.
2. `th:if="${status}"` renders the element only when a model attribute `status` exists. The outer
   `th:if="${timestamp == null}"` skips both on Spring Boot's error pages, whose model has its own
   `status` and `error` (lesson 04, Step 8).
3. `th:text` escapes HTML, so a client name with `<script>` in it is shown as text, never run.
4. `role="status"` and `role="alert"` tell screen readers to announce the message.

---

## Step 3 — The form-backing class `ClientForm`

We never bind a form directly to the `Client` entity. The data binder sets **every** property it finds a
setter for — a user could add `<input name="id" value="1">` in the browser's developer tools. This is
the **mass assignment** problem. A separate form class has only the fields the form shows.

```java
// src/main/java/ee/ta25/billable/client/ClientForm.java
package ee.ta25.billable.client;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Pattern;
import jakarta.validation.constraints.Size;
import java.util.Locale;

/** What the client form sends. Only these fields can be set by the user. */
public class ClientForm {

  @NotBlank(message = "Name is required.")
  @Size(max = 120, message = "Name can be at most 120 characters.")
  private String name;

  @NotBlank(message = "E-mail is required.")
  @Email(message = "Enter a valid e-mail address.")
  @Size(max = 255, message = "E-mail can be at most 255 characters.")
  private String email;

  @Pattern(
      regexp = "^EE\\d{9}$",
      message = "The VAT number must be EE followed by 9 digits, for example EE101234567.")
  private String vatNumber;

  /** Creates a form filled with an existing client's values, for the edit page. */
  public static ClientForm from(Client client) {
    var form = new ClientForm();
    form.setName(client.getName());
    form.setEmail(client.getEmail());
    form.setVatNumber(client.getVatNumber());
    return form;
  }

  public String getName() {
    return name;
  }

  public void setName(String name) {
    this.name = name;
  }

  public String getEmail() {
    return email;
  }

  /** Stores e-mails in lower case, so "Info@Saar.ee" and "info@saar.ee" are the same. */
  public void setEmail(String email) {
    this.email = email == null ? null : email.toLowerCase(Locale.ROOT);
  }

  public String getVatNumber() {
    return vatNumber;
  }

  /** Accepts "ee 101 234 567" and stores "EE101234567". */
  public void setVatNumber(String vatNumber) {
    this.vatNumber =
        vatNumber == null ? null : vatNumber.replace(" ", "").toUpperCase(Locale.ROOT);
  }
}
```

**What happens:**

1. The class has a no-argument constructor (the default one) and setters. The binder creates an empty
   `ClientForm`, then calls `setName(...)`, `setEmail(...)`, `setVatNumber(...)` with the request values.
   Why not a `record`? The create page needs an **empty** form object, and the edit page needs one we
   fill from the entity; a simple mutable class is the most direct fit for that, and it is what the
   Thymeleaf `th:field` examples expect.
2. The setters **normalise** the input. Because binding happens before validation, the rules see the
   cleaned value: `ee 101 234 567` becomes `EE101234567` and passes the pattern.
3. `@NotBlank` fails for `null`, `""` and `"   "`. `@Size(max = 120)` matches the `varchar(120)` column.
   `@Email` checks the basic e-mail format. Every constraint has a `message` in plain English.
4. `@Pattern` treats `null` as valid. Together with step 1 (empty → `null`), that makes the VAT number
   optional but correct when given. `@Pattern` must match the **whole** value; the `^` and `$` make that
   visible to the reader.
5. The imports are `jakarta.validation...`. Old tutorials and many AI answers use `javax.validation...` —
   that package does not exist in Spring Boot 3 or 4.
6. The unique e-mail rule is **not** an annotation here. It needs the database, so the controller checks it
   (step 5).

---

## Step 4 — Repository methods and an `update` method on the entity

Add two query methods to `ClientRepository` (keep `findByNameContainingIgnoreCaseOrderByName`,
`findByEmail` and anything else you already have):

```java
// src/main/java/ee/ta25/billable/client/ClientRepository.java (add inside the interface)
boolean existsByEmailIgnoreCase(String email);

boolean existsByEmailIgnoreCaseAndIdNot(String email, Long id);
```

`ProjectRepository.countByClientId(Long clientId)` exists since lesson 03 — we use it for the delete
check.

Add an `update` method to the `Client` entity, below the constructor:

```java
// src/main/java/ee/ta25/billable/client/Client.java (add inside the class)
public void update(String name, String email, String vatNumber) {
  this.name = name;
  this.email = email;
  this.vatNumber = vatNumber;
}
```

**What happens:**

1. Spring Data reads the **method name** and writes the query for you (a *derived query*).
   `existsByEmailIgnoreCase` becomes roughly
   `select count(*) > 0 from clients where upper(email) = upper(?)`.
2. `...AndIdNot(email, id)` adds `and id <> ?`. This is "unique, but ignore myself": when you edit client
   7, its own e-mail does not count as a duplicate.
3. `countByClientId` (lesson 03) navigates the `client` relation of `Project` and compares its `id`:
   `select count(p1_0.id) from projects p1_0 where p1_0.client_id = ?`.
4. `update(...)` is one method with a meaning ("change the client's details"). The entity also has
   setters from lesson 03, but one call that changes the three form fields together says more, and
   lesson 07 builds on it.

---

## Step 5 — `create` and `store`

Open `ClientController`. Keep `index()`, `show()` and `projectCountsFor()` from lesson 04. The
constructor already receives both repositories. Add the imports and the new methods. The file after
this step (lesson 04 parts shortened to a comment):

```java
// src/main/java/ee/ta25/billable/client/ClientController.java
package ee.ta25.billable.client;

import ee.ta25.billable.common.NotFoundException;
import ee.ta25.billable.project.ProjectRepository;
import jakarta.validation.Valid;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;
// ...plus the lesson 04 imports (HashMap, List, Map, Page, Pageable, PageableDefault, ...)

@Controller
@RequestMapping("/clients")
public class ClientController {

  private final ClientRepository clients;
  private final ProjectRepository projects;

  ClientController(ClientRepository clients, ProjectRepository projects) {
    this.clients = clients;
    this.projects = projects;
  }

  // index(), show() and projectCountsFor() from lesson 04 stay here unchanged.

  @GetMapping("/new")
  public String create(Model model) {
    model.addAttribute("form", new ClientForm());
    return "clients/create";
  }

  @PostMapping
  public String store(
      @Valid @ModelAttribute("form") ClientForm form,
      BindingResult bindingResult,
      RedirectAttributes redirectAttributes) {
    if (form.getEmail() != null && clients.existsByEmailIgnoreCase(form.getEmail())) {
      bindingResult.rejectValue("email", "duplicate", "This e-mail is already used by another client.");
    }
    if (bindingResult.hasErrors()) {
      return "clients/create";
    }

    var client = clients.save(new Client(form.getName(), form.getEmail(), form.getVatNumber()));

    redirectAttributes.addFlashAttribute("status", "Client " + client.getName() + " created.");
    return "redirect:/clients/" + client.getId();
  }

  private Client findClient(long id) {
    return clients
        .findById(id)
        .orElseThrow(() -> new NotFoundException("Client " + id + " not found"));
  }
}
```

`show()` from lesson 04 has the same "find or throw" code inline; you can switch it to `findClient(id)`
too.

**What happens in `create()`:** we put an **empty** `ClientForm` into the model under the name `form`.
The template's `th:object="${form}"` needs that object, even when all fields are empty.

**What happens in `store()`** — follow one submit:

1. The browser sends `POST /clients` with `name`, `email`, `vatNumber`.
2. `@ModelAttribute("form") ClientForm form`: Spring creates a `ClientForm`, runs our `@InitBinder`
   (trim, empty → `null`), and calls the setters with the request values.
3. `@Valid` runs the Jakarta constraints on the form. Violations are **not** thrown as exceptions; they
   are collected in the `BindingResult`.
4. `BindingResult` **must be the parameter directly after** the form. If it is missing or somewhere
   else, Spring throws the errors instead and the user sees a `400 Bad Request` page.
5. We check uniqueness ourselves. `bindingResult.rejectValue("email", "duplicate", "...")` adds an error
   to the field `email`, exactly like a failed annotation would. The arguments are: field name, an error
   code (used for message files, which we do not use), and the default message.
6. **If there are errors**, we return the view name `clients/create`. Spring renders the form again as
   the answer to this POST. The model still contains `form` (with the user's input) and its
   `BindingResult` (with the messages), so the page shows both. Nothing was saved, so a refresh here is
   harmless.
7. **If everything is valid**, we create the entity **explicitly** from the allowed fields and save it.
   `save()` runs `INSERT` and returns the entity with its new `id`.
8. `addFlashAttribute("status", ...)` stores the message for the next request. `"redirect:/clients/7"`
   tells Spring to answer `302 Found` with `Location: /clients/7`. This is Post/Redirect/Get.
9. The browser requests `GET /clients/7`. Spring moves the flash attribute into the model, the layout
   shows the green box.

About the unique check: two people could submit the same e-mail at the same moment and both pass the
`exists` check. The `UNIQUE` constraint on `clients.email` (lesson 03) is the safety net — the second
insert fails with a `DataIntegrityViolationException`. For a single-user app that is fine; lesson 14 looks
at races properly.

Now the templates. The form itself is a **fragment** with two parameters, so create and edit can share
it:

```html
<!-- src/main/resources/templates/clients/form.html -->
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<body>
<form th:fragment="form(action, submitLabel)" th:action="${action}" th:object="${form}" method="post">

  <label for="name">
    Name
    <input type="text" th:field="*{name}" maxlength="120" required
           th:aria-invalid="${#fields.hasErrors('name')} ? 'true'">
    <small th:if="${#fields.hasErrors('name')}" th:errors="*{name}">Name error</small>
  </label>

  <label for="email">
    E-mail
    <input type="email" th:field="*{email}" maxlength="255" required
           th:aria-invalid="${#fields.hasErrors('email')} ? 'true'">
    <small th:if="${#fields.hasErrors('email')}" th:errors="*{email}">E-mail error</small>
  </label>

  <label for="vatNumber">
    VAT number <small>(optional, e.g. EE101234567)</small>
    <input type="text" th:field="*{vatNumber}" maxlength="20"
           th:aria-invalid="${#fields.hasErrors('vatNumber')} ? 'true'">
    <small th:if="${#fields.hasErrors('vatNumber')}" th:errors="*{vatNumber}">VAT error</small>
  </label>

  <button type="submit" th:text="${submitLabel}">Save</button>
</form>
</body>
</html>
```

```html
<!-- src/main/resources/templates/clients/create.html -->
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      th:replace="~{layout :: layout(~{::title}, ~{::main})}">
<head>
  <title>New client · Billable</title>
</head>
<body>
<main class="container">
  <h1>New client</h1>

  <form th:replace="~{clients/form :: form(action=@{/clients}, submitLabel='Create client')}"></form>

  <p><a th:href="@{/clients}">Back to clients</a></p>
</main>
</body>
</html>
```

**What happens in the templates:**

1. The page uses the layout like every page since lesson 01. Inside its `<main>`,
   `th:replace="~{clients/form :: form(...)}"` replaces the placeholder `<form>` with the fragment
   named `form` from `templates/clients/form.html`, and passes the two parameters. `@{/clients}` is a
   link expression, so the URL is correct even if the app runs under a context path.
2. `th:object="${form}"` selects the model attribute `form`. Inside, `*{name}` means "the `name`
   property of the selected object".
3. `th:field="*{name}"` writes three attributes at once: `id="name"`, `name="name"` and
   `value="(current value)"`. After a failed POST the current value is what the user typed, because the
   same form object is in the model.
4. `#fields.hasErrors('name')` is `true` when the `BindingResult` has an error for `name`.
   `th:errors="*{name}"` prints the message(s) for that field.
5. `th:aria-invalid="${...} ? 'true'"` is an *if-then without else*. When there is no error the result is
   `null` and Thymeleaf does not write the attribute at all. With an error it writes
   `aria-invalid="true"`, and Pico.css draws a red border.
6. The fragment file has `<html>` and `<body>` only so that editors show it correctly. Only the `<form>`
   element is used.

**Check it works:**

1. Open `http://localhost:8080/clients/new`.
2. Fill in `Saar OÜ`, ` INFO@saar.ee `, `ee 101 234 567` and submit. You land on the new client's page
   with "Client Saar OÜ created." The e-mail is stored as `info@saar.ee`, the VAT number as `EE101234567`.
3. Press F5. The page reloads without a "Resend form data?" dialog, and the green message is gone.
4. **Test the server-side rules.** The browser checks `required` and `type="email"` itself and would not
   even send the form. For a moment, add `novalidate` to the `<form>` tag in `clients/form.html`. Submit
   an empty form: messages under Name and E-mail. Use the e-mail from step 2 again: "This e-mail is
   already used by another client." VAT `EE123`: the pattern message. Leave VAT empty: no error.
5. Remove `novalidate` again.

**Commit:** `feat(clients): create clients with validation`

---

## Step 6 — `edit` and `update`

Add these two methods to `ClientController`:

```java
// src/main/java/ee/ta25/billable/client/ClientController.java (add below store())
@GetMapping("/{id}/edit")
public String edit(@PathVariable long id, Model model) {
  var client = findClient(id);
  model.addAttribute("client", client);
  model.addAttribute("form", ClientForm.from(client));
  return "clients/edit";
}

@PostMapping("/{id}")
public String update(
    @PathVariable long id,
    @Valid @ModelAttribute("form") ClientForm form,
    BindingResult bindingResult,
    Model model,
    RedirectAttributes redirectAttributes) {
  var client = findClient(id);

  if (form.getEmail() != null && clients.existsByEmailIgnoreCaseAndIdNot(form.getEmail(), id)) {
    bindingResult.rejectValue("email", "duplicate", "This e-mail is already used by another client.");
  }
  if (bindingResult.hasErrors()) {
    model.addAttribute("client", client);
    return "clients/edit";
  }

  client.update(form.getName(), form.getEmail(), form.getVatNumber());
  clients.save(client);

  redirectAttributes.addFlashAttribute("status", "Client updated.");
  return "redirect:/clients/" + id;
}
```

```html
<!-- src/main/resources/templates/clients/edit.html -->
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      th:replace="~{layout :: layout(~{::title}, ~{::main})}">
<head>
  <title th:text="|Edit ${client.name} · Billable|">Edit client · Billable</title>
</head>
<body>
<main class="container">
  <h1 th:text="|Edit ${client.name}|">Edit client</h1>

  <form th:replace="~{clients/form :: form(action=@{/clients/{id}(id=${client.id})}, submitLabel='Save changes')}"></form>

  <p><a th:href="@{/clients/{id}(id=${client.id})}">Cancel</a></p>
</main>
</body>
</html>
```

**What happens:**

1. `edit()` loads the client (404 if unknown) and fills a `ClientForm` from it with `ClientForm.from()`.
   The template shows the stored values.
2. `POST /clients/7` calls `update()`. The same binding and validation as in `store()` happen.
3. The unique check uses `existsByEmailIgnoreCaseAndIdNot(email, 7)`: other clients' e-mails are
   duplicates, client 7's own e-mail is not.
4. On errors we must add `client` to the model again, because `edit.html` uses `client.name` and
   `client.id`. (`form` is already in the model — Spring put it there because of `@ModelAttribute`.)
5. On success, `client.update(...)` changes the entity in memory and `clients.save(client)` writes it.
   Because the controller has no transaction (`open-in-view` is `false`), the `client` we loaded is
   *detached*: `save()` merges it (Hibernate runs a `SELECT`, then an `UPDATE`). In lesson 07 we move this
   into a `@Transactional` service method, where the `save()` call is no longer needed.
6. `|Edit ${client.name}|` is a *literal substitution*: a string with `${...}` parts inside the `|`
   characters.
7. `@{/clients/{id}(id=${client.id})}` builds `/clients/7`. `{id}` is a placeholder, and `(id=...)` fills it.

**Check it works:**

1. Open `http://localhost:8080/clients/1/edit`. The fields show the stored values.
2. Change only the name and save. It works — the unchanged e-mail is not reported as a duplicate.
3. Change the e-mail to another client's e-mail. You see the duplicate message, and your typed value is
   still in the field.

**Commit:** `feat(clients): edit clients, unique e-mail ignores self`

---

## Step 7 — Delete with confirmation and a friendly FK error

Add the delete method to `ClientController`:

```java
// src/main/java/ee/ta25/billable/client/ClientController.java (add below update())
@PostMapping("/{id}/delete")
public String destroy(@PathVariable long id, RedirectAttributes redirectAttributes) {
  var client = findClient(id);
  long projectCount = projects.countByClientId(id);

  if (projectCount > 0) {
    redirectAttributes.addFlashAttribute(
        "error",
        client.getName()
            + " still has "
            + projectCount
            + " project(s). Delete or archive them first.");
    return "redirect:/clients/" + id;
  }

  clients.delete(client);

  redirectAttributes.addFlashAttribute("status", "Client " + client.getName() + " deleted.");
  return "redirect:/clients";
}
```

Add the buttons to the client page `templates/clients/show.html`, below the client's details and above
the project list:

```html
<!-- src/main/resources/templates/clients/show.html (below the client details) -->
<div class="grid">
  <a th:href="@{/clients/{id}/edit(id=${client.id})}" role="button" class="secondary">Edit</a>

  <form th:action="@{/clients/{id}/delete(id=${client.id})}" method="post"
        th:data-confirm="|Delete client ${client.name}? This cannot be undone.|"
        onsubmit="return confirm(this.dataset.confirm)">
    <button type="submit" class="contrast">Delete</button>
  </form>
</div>
```

And on the list `templates/clients/index.html`, below the `<h1>`:

```html
<!-- src/main/resources/templates/clients/index.html (below the h1) -->
<p><a th:href="@{/clients/new}" role="button">New client</a></p>
```

**What happens:**

1. `countByClientId(id)` runs one `count(*)` query. If the client has projects we do **not** try to
   delete; we redirect back to the client page with the red `error` flash.
2. Without that check, `clients.delete(client)` would fail in the database because of the restrict
   foreign key on `projects.client_id`. Spring would turn it into a
   `DataIntegrityViolationException` and the user would get a `500` error page:
   ```
   ERROR: update or delete on table "clients" violates foreign key constraint
          "..." on table "projects"
   ```
   The foreign key stays as the safety net; the check turns the rule into a helpful message.
3. Delete is a **form with POST**, not a link. Crawlers and chat link previews only follow GET links, so
   they can never delete anything.
4. `th:data-confirm` writes a normal, correctly escaped `data-confirm` attribute. `onsubmit` shows the
   browser's OK/Cancel dialog with that text; `return false` (Cancel) stops the submit. Putting the name
   directly inside the JavaScript would break for names with quotes, like `O'Brien`.
5. `role="button"` lets Pico.css draw the Edit link as a button. `class="grid"` places the two side by side.

**Check it works:**

1. `/clients` shows "New client"; a client page shows "Edit" and "Delete".
2. Create a client without projects, press Delete, then **Cancel** — nothing happens. Press Delete, **OK**
   — you land on `/clients` with "Client … deleted."
3. Open a seeded client with projects and press Delete → OK. You stay on the page with the red message.
   Nothing was deleted.

**Commit:** `feat(clients): delete clients with confirmation and project check`

---

## Step 8 — CSRF protection with Spring Security

**Why:** in the Laravel track every form carries a hidden CSRF token. Look at the HTML of your form now
(right click → View page source): there is **no** token. Billable has no login, so it accepts every
request. Any web page that a user on the same network opens could make their browser send a POST to
Billable (for example "delete client 12"), and Billable would do it. "No login" does not make that safe
(README, concept 7).

In Spring, CSRF protection is part of **Spring Security**. We add it with one rule that lets everyone in:
no login page, no users — only the CSRF check.

Add two dependencies to `pom.xml` (no `<version>` — the Boot parent manages them):

```xml
<!-- pom.xml (inside <dependencies>) -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security-test</artifactId>
  <scope>test</scope>
</dependency>
```

Reload the Maven project in IntelliJ, then create the configuration:

```java
// src/main/java/ee/ta25/billable/common/SecurityConfig.java
package ee.ta25.billable.common;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

/** No login (Billable is single-user), but CSRF protection stays on for every form. */
@Configuration
public class SecurityConfig {

  @Bean
  SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(auth -> auth.anyRequest().permitAll());
    return http.build();
  }
}
```

Restart the application.

**What happens:**

1. With `spring-boot-starter-security` on the classpath, every request first goes through Spring
   Security's filter chain, before `DispatcherServlet`.
2. Our `SecurityFilterChain` bean replaces Boot's default one. The default would ask for a login on
   every page; `anyRequest().permitAll()` lets every request in. We do not call `formLogin()` or
   `httpBasic()`, so there is no login page and no browser password dialog.
3. We do not switch CSRF off, so it stays **on** — that is Spring Security's default. Every POST must
   carry a valid token, otherwise the answer is `403 Forbidden`. GET requests need no token.
4. `th:action` adds the token for us. When Thymeleaf renders a `<form>` with `th:action` and
   `method="post"`, it asks Spring Security for the token and writes a hidden field
   `<input type="hidden" name="_csrf" value="…">` into the form. That works for the form fragment
   (`th:action="${action}"`) and for the delete button form alike. A plain `action="/clients"` without
   `th:` gets **no** token — one more reason to always write `th:action="@{...}"`.
5. The token lives in the HTTP session. An attacker's page cannot read it, so it cannot build a valid
   POST.
6. In the start-up log you now see `Using generated security password: …`. Spring Boot still creates
   a default user, because we did not define our own users. Nothing asks for that password: ignore it.
7. Spring Security also adds a few safe response headers to every page (for example
   `X-Frame-Options: DENY` and `X-Content-Type-Options: nosniff`).

**Check it works:**

1. View the source of `/clients/new` and of a client page: each POST form has a hidden `_csrf` input.
   A `method="get"` form, such as the search form from lesson 04's independent work, has none — correct.
2. Create, edit and delete still work as before.
3. Send a POST without a token from a terminal:
   `curl -i -X POST http://localhost:8080/clients/1/delete`. The answer is `HTTP/1.1 403` and nothing is
   deleted.

From now on, tests that send a POST need a token too: Spring Security's test support gives it with
`.with(csrf())` (lesson 12).

**Commit:** `feat(security): add Spring Security for CSRF protection, no login`

---

## Step 9 — Format and final check

```bash
./mvnw spotless:apply
./mvnw spotless:check
./mvnw spring-boot:run
```

**Check it works:** walk through the whole cycle once: create → green message → edit → duplicate e-mail →
error → fix → save → delete a client with projects (red) and one without (green).

**Commit:** `style: apply spotless` (only if Spotless changed something).

---

## Troubleshooting

| What you see | Cause | Fix |
|---|---|---|
| `Neither BindingResult nor plain target object for bean name 'form' available as request attribute` | The template uses `th:object="${form}"` but the controller did not add `form` to the model | `model.addAttribute("form", new ClientForm())` in `create()` / `ClientForm.from(...)` in `edit()` |
| Whitelabel page `400 Bad Request`, log shows `MethodArgumentNotValidException` | `BindingResult` is missing or is not directly after the `@ModelAttribute` parameter | Put `BindingResult bindingResult` right after the form parameter |
| `Request method 'POST' is not supported` (405) | The form posts to a URL without a `@PostMapping`, e.g. a typo in `/delete` | Compare `th:action` with the mapping |
| Empty VAT field shows the pattern error | `FormBindingAdvice` is missing, so `""` is validated | Step 1; check the class is in `ee.ta25.billable.common` (inside the scanned package) |
| `package javax.validation.constraints does not exist` | Old tutorial code | Use `jakarta.validation.constraints` |
| Constraints are ignored completely | `@Valid` missing, or `spring-boot-starter-validation` not in `pom.xml` | Add both |
| Green message never appears | `model.addAttribute` used instead of `redirectAttributes.addFlashAttribute`, or the page does not use the layout (`th:replace="~{layout :: layout(~{::title}, ~{::main})}"` on `<html>`) | Use flash attributes; wrap the page in the layout |
| `EL1007E: Property or field 'name' cannot be found on null` in `edit.html` after a failed update | `client` not added to the model in the error branch | `model.addAttribute("client", client)` before `return "clients/edit"` |
| `DataIntegrityViolationException ... violates foreign key constraint` | Deleting a client with projects without the check | Step 7 |
| `Error resolving template [clients/form]` | Fragment file name or path differs | The file must be `templates/clients/form.html` with `th:fragment="form(action, submitLabel)"` |
| `403 Forbidden` (Whitelabel page) after submitting a form | The form has no `_csrf` field: it uses a plain `action="..."`, or the hidden field was removed | Use `th:action="@{...}"` with `method="post"` (Step 8) |
| Every page asks for a user name and password, or redirects to `/login` | `SecurityConfig` is not found (wrong package) or has no `@Configuration`, so Boot's default security applies | The class must be in `ee.ta25.billable.common` with `@Configuration` and the `@Bean` method (Step 8) |
| Log line `Using generated security password: …` | Boot creates a default user because we defined none | Harmless; no page asks for it (Step 8) |

---

## Recap

- `common/FormBindingAdvice.java` — trims all strings, empty → `null`, for every controller.
- `client/ClientForm.java` — form-backing class with Jakarta constraints and normalising setters.
- `client/ClientRepository.java` — `existsByEmailIgnoreCase`, `existsByEmailIgnoreCaseAndIdNot`.
- `project/ProjectRepository.java` — `countByClientId`.
- `client/Client.java` — `update(name, email, vatNumber)`.
- `client/ClientController.java` — `create`, `store`, `edit`, `update`, `destroy`; PRG, flash, unique
  check with `rejectValue`, project check before delete.
- `templates/clients/form.html` (fragment), `create.html`, `edit.html`; buttons in `index.html`,
  `show.html`; flash boxes in `layout.html`.
- `pom.xml` — `spring-boot-starter-security` and `spring-boot-starter-security-test`.
- `common/SecurityConfig.java` — `permitAll()` for every request, CSRF left on (the default);
  `th:action` adds the hidden `_csrf` field to every POST form.

---

## Independent work — solutions

> **The tasks are not here.** The task descriptions (Basic / Intermediate / Advanced, with acceptance
> criteria) are in this lesson's [README.md → Independent work (~2 h)](README.md#independent-work-2-h).
> Read them there first and build the features in your project. This section only contains the
> solutions — try each task yourself, then open a solution to compare or when you are stuck.

<details>
<summary><strong>Basic — full CRUD for projects</strong></summary>

This builds on the lesson 03 entity `project.Project`: `client` (`@ManyToOne(fetch = LAZY)`), `name`,
`billingType` (`@Enumerated(EnumType.STRING)`, stored lowercase via `@EnumeratedValue`), `hourlyRateCents`, `fixedPriceCents`, `budgetCapCents`
(all `Long` — the columns are nullable `BIGINT`, and a `Long` can hold `null`), `roundingMinutes`
(`short` field; `getRoundingMinutes()` returns `int`, and `setRoundingMinutes(int)` rejects anything but
1, 6, 15), `archivedAt`, and `BillingType` with `HOURLY`, `FIXED`, `CAPPED` and `label()`.

**1. Entity methods.** Add a constructor and an `update` method to `Project`:

```java
// src/main/java/ee/ta25/billable/project/Project.java (add inside the class)
public Project(Client client) {
  this.client = client;
}

public void update(
    String name,
    BillingType billingType,
    Long hourlyRateCents,
    Long fixedPriceCents,
    Long budgetCapCents,
    int roundingMinutes) {
  this.name = name;
  this.billingType = billingType;
  this.hourlyRateCents = hourlyRateCents;
  this.fixedPriceCents = fixedPriceCents;
  this.budgetCapCents = budgetCapCents;
  setRoundingMinutes(roundingMinutes); // lesson 03 setter: checks 1/6/15 and stores a short
}
```

The constructor `Project(Client)` creates an empty project for one client; `update(...)` then sets all
form fields in one call (for a new project and for an edit). The lesson 03 constructor
`Project(Client, String, BillingType, int)` stays for the seeder.

**2. The form class.** The conditional rules depend on **two** fields, so a single-field annotation such
as `@NotBlank` cannot express them. We use `@AssertTrue` on methods (lesson 06 uses the same tool for
time entries): Jakarta Validation calls them like getters and reports an error when they return `false`.

```java
// src/main/java/ee/ta25/billable/project/ProjectForm.java
package ee.ta25.billable.project;

import jakarta.validation.constraints.AssertTrue;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Pattern;
import jakarta.validation.constraints.Size;
import java.math.BigDecimal;
import java.util.Set;

/** What the project form sends. Money fields are text in euros, e.g. "85.50" or "85,50". */
public class ProjectForm {

  /** 1–7 digits, optionally followed by a dot or comma and 1–2 digits. */
  private static final String EUROS = "^\\d{1,7}([.,]\\d{1,2})?$";

  private static final Set<Integer> ROUNDING_OPTIONS = Set.of(1, 6, 15);

  @NotBlank(message = "Name is required.")
  @Size(max = 120, message = "Name can be at most 120 characters.")
  private String name;

  @NotNull(message = "Choose a billing type.")
  private BillingType billingType = BillingType.HOURLY;

  @Pattern(regexp = EUROS, message = "Use a format like 85 or 85.50.")
  private String hourlyRate;

  @Pattern(regexp = EUROS, message = "Use a format like 1200 or 1200.00.")
  private String fixedPrice;

  @Pattern(regexp = EUROS, message = "Use a format like 3000 or 3000.00.")
  private String budgetCap;

  @NotNull(message = "Choose a rounding.")
  private Integer roundingMinutes = 15;

  public static ProjectForm from(Project project) {
    var form = new ProjectForm();
    form.name = project.getName();
    form.billingType = project.getBillingType();
    form.hourlyRate = toEuros(project.getHourlyRateCents());
    form.fixedPrice = toEuros(project.getFixedPriceCents());
    form.budgetCap = toEuros(project.getBudgetCapCents());
    form.roundingMinutes = project.getRoundingMinutes();
    return form;
  }

  // Rules that depend on the billing type. Each returns true when the rule does not apply.

  @AssertTrue(message = "Hourly and capped projects need an hourly rate.")
  public boolean isHourlyRateGiven() {
    return billingType == null || !usesHourlyRate() || hourlyRate != null;
  }

  @AssertTrue(message = "Fixed price projects need a price.")
  public boolean isFixedPriceGiven() {
    return billingType != BillingType.FIXED || fixedPrice != null;
  }

  @AssertTrue(message = "Capped projects need a budget cap.")
  public boolean isBudgetCapGiven() {
    return billingType != BillingType.CAPPED || budgetCap != null;
  }

  @AssertTrue(message = "Rounding must be 1, 6 or 15 minutes.")
  public boolean isRoundingAllowed() {
    return roundingMinutes == null || ROUNDING_OPTIONS.contains(roundingMinutes);
  }

  /** Copies the form into the entity. Fields that do not apply are stored as null. */
  public void applyTo(Project project) {
    project.update(
        name,
        billingType,
        usesHourlyRate() ? toCents(hourlyRate) : null,
        billingType == BillingType.FIXED ? toCents(fixedPrice) : null,
        billingType == BillingType.CAPPED ? toCents(budgetCap) : null,
        roundingMinutes);
  }

  private boolean usesHourlyRate() {
    return switch (billingType) {
      case HOURLY, CAPPED -> true;
      case FIXED -> false;
    };
  }

  /** "85,5" → 8550. The text must already match EUROS. Only integers are used, no floating point. */
  static long toCents(String euros) {
    var normalized = euros.replace(',', '.');
    var dot = normalized.indexOf('.');
    if (dot < 0) {
      return Long.parseLong(normalized) * 100;
    }
    var whole = normalized.substring(0, dot);
    var fraction = (normalized.substring(dot + 1) + "00").substring(0, 2);
    return Long.parseLong(whole) * 100 + Long.parseLong(fraction);
  }

  /** 8550 → "85.50", -5 → "-0.05", null → null. */
  static String toEuros(Long cents) {
    return cents == null ? null : BigDecimal.valueOf(cents, 2).toPlainString();
  }

  // Getters and setters for all six fields (the binder and th:field need them).

  public String getName() {
    return name;
  }

  public void setName(String name) {
    this.name = name;
  }

  public BillingType getBillingType() {
    return billingType;
  }

  public void setBillingType(BillingType billingType) {
    this.billingType = billingType;
  }

  public String getHourlyRate() {
    return hourlyRate;
  }

  public void setHourlyRate(String hourlyRate) {
    this.hourlyRate = hourlyRate;
  }

  public String getFixedPrice() {
    return fixedPrice;
  }

  public void setFixedPrice(String fixedPrice) {
    this.fixedPrice = fixedPrice;
  }

  public String getBudgetCap() {
    return budgetCap;
  }

  public void setBudgetCap(String budgetCap) {
    this.budgetCap = budgetCap;
  }

  public Integer getRoundingMinutes() {
    return roundingMinutes;
  }

  public void setRoundingMinutes(Integer roundingMinutes) {
    this.roundingMinutes = roundingMinutes;
  }
}
```

Notes:

- `@AssertTrue` on `isHourlyRateGiven()`: the `is` prefix is removed, so the error is stored under the
  name `hourlyRateGiven`, not `hourlyRate`. That is why the template shows `th:errors="*{hourlyRateGiven}"`
  under the hourly rate input, next to the `hourlyRate` format error. The controller needs no extra call:
  `@Valid` runs these methods together with the field annotations.
- The methods return `true` when the rule does not apply, or when `billingType` is missing (`@NotNull`
  already reports that; one message is enough).
- `toCents("85,5")`: comma → dot, `whole = "85"`, `fraction = ("5" + "00").substring(0, 2) = "50"`,
  result `8550` as a `long` (R1: money is `long` cents in Java). The 7-digit limit in `EUROS` is only a
  sanity check against typing mistakes; a `long` could hold far more.
- An alternative without string splitting is `new BigDecimal(normalized).movePointRight(2).longValueExact()`
  — `BigDecimal` is exact decimal arithmetic. Never use `Double.parseDouble`. Lesson 10 replaces this
  with a `Money` type.
- `toEuros` uses `BigDecimal.valueOf(cents, 2)`: "`cents` with the decimal point moved 2 places", exact.
  `toPlainString()` gives `85.50`, and `-5` gives `-0.05`. (Hand-written `cents / 100` and `cents % 100`
  would give `0.-5`, because the remainder of a negative number is negative.)
- The switch expression over the enum has no `default`: if someone adds a fourth billing type, the
  compiler reports that this switch is not complete.

**3. Repository method for the list** (active projects first). Add to `ProjectRepository`:

```java
// src/main/java/ee/ta25/billable/project/ProjectRepository.java (add inside the interface)
@Query(
    """
    select p from Project p
    where p.client.id = :clientId
    order by case when p.archivedAt is null then 0 else 1 end, p.name
    """)
List<Project> findByClientIdActiveFirst(@Param("clientId") Long clientId);
```

(Imports: `org.springframework.data.jpa.repository.Query`, `org.springframework.data.repository.query.Param`,
`java.util.List`.) Use it in your lesson 04 `index()` instead of the old method if you want the order now;
the intermediate task needs it.

**4. Controller.** The new methods (keep `index()` and `show()` from lesson 04):

```java
// src/main/java/ee/ta25/billable/project/ProjectController.java
package ee.ta25.billable.project;

import ee.ta25.billable.client.Client;
import ee.ta25.billable.client.ClientRepository;
import ee.ta25.billable.common.NotFoundException;
import jakarta.validation.Valid;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

@Controller
public class ProjectController {

  private final ClientRepository clients;
  private final ProjectRepository projects;

  ProjectController(ClientRepository clients, ProjectRepository projects) {
    this.clients = clients;
    this.projects = projects;
  }

  /** Added to the model of every request handled by this controller. */
  @ModelAttribute("billingTypes")
  public BillingType[] billingTypes() {
    return BillingType.values();
  }

  // index() (GET /clients/{clientId}/projects) and show() (GET /projects/{id}) from lesson 04.

  @GetMapping("/clients/{clientId}/projects/new")
  public String create(@PathVariable long clientId, Model model) {
    model.addAttribute("client", findClient(clientId));
    model.addAttribute("form", new ProjectForm());
    return "projects/create";
  }

  @PostMapping("/clients/{clientId}/projects")
  public String store(
      @PathVariable long clientId,
      @Valid @ModelAttribute("form") ProjectForm form,
      BindingResult bindingResult,
      Model model,
      RedirectAttributes redirectAttributes) {
    var client = findClient(clientId);
    if (bindingResult.hasErrors()) {
      model.addAttribute("client", client);
      return "projects/create";
    }

    var project = new Project(client);
    form.applyTo(project);
    projects.save(project);

    redirectAttributes.addFlashAttribute("status", "Project " + project.getName() + " created.");
    return "redirect:/projects/" + project.getId();
  }

  @GetMapping("/projects/{id}/edit")
  public String edit(@PathVariable long id, Model model) {
    var project = findProject(id);
    model.addAttribute("project", project);
    model.addAttribute("form", ProjectForm.from(project));
    return "projects/edit";
  }

  @PostMapping("/projects/{id}")
  public String update(
      @PathVariable long id,
      @Valid @ModelAttribute("form") ProjectForm form,
      BindingResult bindingResult,
      Model model,
      RedirectAttributes redirectAttributes) {
    var project = findProject(id);
    if (bindingResult.hasErrors()) {
      model.addAttribute("project", project);
      return "projects/edit";
    }

    form.applyTo(project);
    projects.save(project);

    redirectAttributes.addFlashAttribute("status", "Project updated.");
    return "redirect:/projects/" + id;
  }

  @PostMapping("/projects/{id}/delete")
  public String destroy(@PathVariable long id, RedirectAttributes redirectAttributes) {
    var project = findProject(id);
    var clientId = project.getClient().getId();
    projects.delete(project);

    redirectAttributes.addFlashAttribute("status", "Project " + project.getName() + " deleted.");
    return "redirect:/clients/" + clientId;
  }

  private Client findClient(long id) {
    return clients
        .findById(id)
        .orElseThrow(() -> new NotFoundException("Client " + id + " not found"));
  }

  private Project findProject(long id) {
    return projects
        .findById(id)
        .orElseThrow(() -> new NotFoundException("Project " + id + " not found"));
  }
}
```

- `@ModelAttribute("billingTypes")` on a **method** runs before every handler of this controller and puts
  the enum values into the model — also in the error branches, so we cannot forget it.
- `project.getClient().getId()` does not load the client: Hibernate knows the id of a lazy proxy without
  a query. Calling `getClient().getName()` here would throw `LazyInitializationException`, because there is
  no open session.
- The client comes from the **URL**, not from the form. The user cannot move a project to another client
  by editing the HTML.

**5. Templates.**

```html
<!-- src/main/resources/templates/projects/form.html -->
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<body>
<form th:fragment="form(action, submitLabel)" th:action="${action}" th:object="${form}" method="post">

  <label for="name">
    Name
    <input type="text" th:field="*{name}" maxlength="120" required
           th:aria-invalid="${#fields.hasErrors('name')} ? 'true'">
    <small th:if="${#fields.hasErrors('name')}" th:errors="*{name}">Error</small>
  </label>

  <label for="billingType">
    Billing type
    <select th:field="*{billingType}" th:aria-invalid="${#fields.hasErrors('billingType')} ? 'true'">
      <option th:each="type : ${billingTypes}" th:value="${type}" th:text="${type.label()}">Hourly</option>
    </select>
    <small th:if="${#fields.hasErrors('billingType')}" th:errors="*{billingType}">Error</small>
  </label>

  <div class="grid">
    <label for="hourlyRate">
      Hourly rate (€) <small>hourly and capped</small>
      <input type="text" inputmode="decimal" th:field="*{hourlyRate}"
             th:aria-invalid="${#fields.hasErrors('hourlyRate') or #fields.hasErrors('hourlyRateGiven')} ? 'true'">
      <small th:if="${#fields.hasErrors('hourlyRate')}" th:errors="*{hourlyRate}">Error</small>
      <small th:if="${#fields.hasErrors('hourlyRateGiven')}" th:errors="*{hourlyRateGiven}">Error</small>
    </label>

    <label for="fixedPrice">
      Fixed price (€) <small>fixed only</small>
      <input type="text" inputmode="decimal" th:field="*{fixedPrice}"
             th:aria-invalid="${#fields.hasErrors('fixedPrice') or #fields.hasErrors('fixedPriceGiven')} ? 'true'">
      <small th:if="${#fields.hasErrors('fixedPrice')}" th:errors="*{fixedPrice}">Error</small>
      <small th:if="${#fields.hasErrors('fixedPriceGiven')}" th:errors="*{fixedPriceGiven}">Error</small>
    </label>

    <label for="budgetCap">
      Budget cap (€) <small>capped only</small>
      <input type="text" inputmode="decimal" th:field="*{budgetCap}"
             th:aria-invalid="${#fields.hasErrors('budgetCap') or #fields.hasErrors('budgetCapGiven')} ? 'true'">
      <small th:if="${#fields.hasErrors('budgetCap')}" th:errors="*{budgetCap}">Error</small>
      <small th:if="${#fields.hasErrors('budgetCapGiven')}" th:errors="*{budgetCapGiven}">Error</small>
    </label>
  </div>

  <label for="roundingMinutes">
    Round each entry up to
    <select th:field="*{roundingMinutes}">
      <option th:each="minutes : ${ {1, 6, 15} }" th:value="${minutes}" th:text="|${minutes} min|">15 min</option>
    </select>
    <small th:if="${#fields.hasErrors('roundingMinutes')}" th:errors="*{roundingMinutes}">Error</small>
    <small th:if="${#fields.hasErrors('roundingAllowed')}" th:errors="*{roundingAllowed}">Error</small>
  </label>

  <button type="submit" th:text="${submitLabel}">Save</button>
</form>
</body>
</html>
```

`th:field` on a `<select>` marks the option whose value equals the current field value as `selected`.
`${ {1, 6, 15} }` is an inline list in Spring's expression language. `type="text" inputmode="decimal"`
instead of `type="number"`, because a number input rejects `85,50` in some browsers.

```html
<!-- src/main/resources/templates/projects/create.html -->
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      th:replace="~{layout :: layout(~{::title}, ~{::main})}">
<head>
  <title>New project · Billable</title>
</head>
<body>
<main class="container">
  <h1 th:text="|New project for ${client.name}|">New project</h1>
  <form th:replace="~{projects/form :: form(action=@{/clients/{id}/projects(id=${client.id})}, submitLabel='Create project')}"></form>
  <p><a th:href="@{/clients/{id}(id=${client.id})}">Back</a></p>
</main>
</body>
</html>
```

```html
<!-- src/main/resources/templates/projects/edit.html -->
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      th:replace="~{layout :: layout(~{::title}, ~{::main})}">
<head>
  <title th:text="|Edit ${project.name} · Billable|">Edit project · Billable</title>
</head>
<body>
<main class="container">
  <h1 th:text="|Edit ${project.name}|">Edit project</h1>
  <form th:replace="~{projects/form :: form(action=@{/projects/{id}(id=${project.id})}, submitLabel='Save changes')}"></form>
  <p><a th:href="@{/projects/{id}(id=${project.id})}">Cancel</a></p>
</main>
</body>
</html>
```

On the client page add
`<a th:href="@{/clients/{id}/projects/new(id=${client.id})}" role="button">New project</a>` above the
project list. On the project page add:

```html
<!-- src/main/resources/templates/projects/show.html (below the details) -->
<div class="grid">
  <a th:href="@{/projects/{id}/edit(id=${project.id})}" role="button" class="secondary">Edit</a>
  <form th:action="@{/projects/{id}/delete(id=${project.id})}" method="post"
        th:data-confirm="|Delete project ${project.name}?|"
        onsubmit="return confirm(this.dataset.confirm)">
    <button type="submit" class="contrast">Delete</button>
  </form>
</div>
```

**Check:** a capped project without a cap → "Capped projects need a budget cap." under the cap field.
Rate `85,5` → the project page shows it with `@formatters.euros(...)` and the database has `8550`
(check with `docker compose exec postgres psql -U billable -d billable -c "select name, hourly_rate_cents from projects"`).
Change a project to fixed → `hourly_rate_cents` becomes `null`.

</details>

<details>
<summary><strong>Intermediate — archive instead of delete</strong></summary>

**Entity methods.** `Project` already has `isArchived()` and `archive(LocalDateTime when)` from lesson
03 (the seeder uses it). Add a short form of `archive` and the opposite action, below them:

```java
// src/main/java/ee/ta25/billable/project/Project.java (add below archive(LocalDateTime))
public void archive() {
  archive(LocalDateTime.now());
}

public void unarchive() {
  this.archivedAt = null;
}
```

`LocalDateTime.now()` reads the system clock directly. That makes it hard to test; in lesson 09/10 we
inject a `Clock` instead. For now it is fine.

**Controller** — remove `destroy()` and add:

```java
// src/main/java/ee/ta25/billable/project/ProjectController.java (replace destroy())
@PostMapping("/projects/{id}/archive")
public String archive(@PathVariable long id, RedirectAttributes redirectAttributes) {
  var project = findProject(id);
  project.archive();
  projects.save(project);
  redirectAttributes.addFlashAttribute("status", "Project " + project.getName() + " archived.");
  return "redirect:/projects/" + id;
}

@PostMapping("/projects/{id}/unarchive")
public String unarchive(@PathVariable long id, RedirectAttributes redirectAttributes) {
  var project = findProject(id);
  project.unarchive();
  projects.save(project);
  redirectAttributes.addFlashAttribute(
      "status", "Project " + project.getName() + " is active again.");
  return "redirect:/projects/" + id;
}
```

Use `findByClientIdActiveFirst(clientId)` (basic task, step 3) wherever a client's projects are listed.

**Template** — replace the delete form on `projects/show.html`:

```html
<!-- src/main/resources/templates/projects/show.html (replace the delete form) -->
<th:block th:if="${project.archived}">
  <p><mark th:text="|Archived on ${#temporals.format(project.archivedAt, 'dd.MM.yyyy')}|">Archived</mark></p>
  <form th:action="@{/projects/{id}/unarchive(id=${project.id})}" method="post">
    <button type="submit" class="secondary">Unarchive</button>
  </form>
</th:block>
<form th:unless="${project.archived}" th:action="@{/projects/{id}/archive(id=${project.id})}" method="post"
      th:data-confirm="|Archive project ${project.name}?|"
      onsubmit="return confirm(this.dataset.confirm)">
  <button type="submit" class="contrast">Archive</button>
</form>
```

`${project.archived}` calls `isArchived()` — for `boolean` properties the getter is `isX()`.
`#temporals` formats `java.time` values; it is built into Thymeleaf 3.1. In the list, add
`<small th:if="${project.archived}">(archived)</small>` next to the name.

</details>

<details>
<summary><strong>Advanced — reusable <code>@EstonianVatNumber</code> constraint</strong></summary>

A custom constraint has two parts: the **annotation** (what you write on the field) and the
**validator** (the code that checks).

```java
// src/main/java/ee/ta25/billable/common/EstonianVatNumber.java
package ee.ta25.billable.common;

import static java.lang.annotation.ElementType.FIELD;
import static java.lang.annotation.ElementType.PARAMETER;
import static java.lang.annotation.RetentionPolicy.RUNTIME;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;
import java.lang.annotation.Documented;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;

/** An Estonian VAT number (KMKR number): "EE" followed by exactly 9 digits. null is valid. */
@Documented
@Constraint(validatedBy = EstonianVatNumberValidator.class)
@Target({FIELD, PARAMETER})
@Retention(RUNTIME)
public @interface EstonianVatNumber {

  String message() default "must be EE followed by 9 digits, for example EE101234567";

  Class<?>[] groups() default {};

  Class<? extends Payload>[] payload() default {};
}
```

```java
// src/main/java/ee/ta25/billable/common/EstonianVatNumberValidator.java
package ee.ta25.billable.common;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import java.util.regex.Pattern;

public class EstonianVatNumberValidator implements ConstraintValidator<EstonianVatNumber, String> {

  private static final Pattern FORMAT = Pattern.compile("^EE\\d{9}$");

  @Override
  public boolean isValid(String value, ConstraintValidatorContext context) {
    if (value == null) {
      return true; // optional field: combine with @NotNull if it must be present
    }
    return FORMAT.matcher(value).matches();
  }
}
```

Use it in `ClientForm` instead of `@Pattern`:

```java
// src/main/java/ee/ta25/billable/client/ClientForm.java (replace the @Pattern on vatNumber)
@EstonianVatNumber(message = "The VAT number must be EE followed by 9 digits, for example EE101234567.")
private String vatNumber;
```

and import `ee.ta25.billable.common.EstonianVatNumber`.

- `@Constraint(validatedBy = ...)` links the annotation to its validator. Hibernate Validator creates the
  validator for you.
- `message`, `groups` and `payload` are **required** members of every constraint annotation — without
  them you get `HV000074: ... contains Constraint annotation, but does not contain a message parameter`.
- `null` returns `true`, following the convention of the built-in constraints: "no value" is the job of
  `@NotNull` / `@NotBlank`.
- `Pattern.compile` runs once (a `static final` field), not on every call.
- The validator is a plain class. In lesson 11 you can unit-test it with `new EstonianVatNumberValidator()`
  and no Spring at all.

</details>
