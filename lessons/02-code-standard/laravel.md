# 02 — An agreed code standard · Laravel

[← Concepts](README.md) · Starting point: end of lesson 01 · Estimated time in class: 2 h

---

## What we build today

- **Laravel Pint** configured with `pint.json` (preset `laravel`) and run over the whole project.
- An **`.editorconfig`** adjusted for Billable.
- **`CODESTYLE.md`** — the group's agreement, with the date and the results of the vote.
- A **GitHub Actions** workflow that installs dependencies and runs `pint --test` on every push and
  pull request.
- A **pull request** `chore/code-standard` that we deliberately break (CI red), fix (CI green) and
  merge.

At the end, the **Actions** tab of your GitHub repository shows a red run and a green run, and `main`
contains all of the above.

We work on a **branch** from the start today — this lesson is where we stop committing directly to
`main`.

---

## Step 1 — Start a branch

**Why:** everything we add today reaches `main` through a pull request, so you see the full team
workflow once, end to end.

```bash
git switch main
git pull
git status
git switch -c chore/code-standard
```

**What happens:**

1. `git pull` makes sure your local `main` has everything from GitHub (for example if you worked on
   another computer).
2. `git status` must say `nothing to commit, working tree clean`. If not, commit or stash your changes
   first — a branch should start clean.
3. `git switch -c chore/code-standard` creates a new branch and switches to it. The name follows the
   pattern `type/short-description`, with the same types as commit messages.

**Check it works:** `git branch` shows `* chore/code-standard`.

---

## Step 2 — Run Pint for the first time

**Why:** Pint is already installed — every new Laravel project includes it as a development
dependency. Before configuring anything, let us see what it thinks of our code.

Check that it is there:

```bash
grep pint composer.json
```

You see `"laravel/pint": "^1.…"` in the `require-dev` section.

Run it in **check mode** (it reports, but does not change files):

```bash
./vendor/bin/pint --test
```

On Windows PowerShell use `vendor\bin\pint --test` (or `php vendor/bin/pint --test`).

**What happens:**

1. Pint reads every PHP file in the project (it skips `vendor/`, `storage/` and other generated
   folders automatically).
2. It compares each file with the rules of the preset. Without a `pint.json`, the preset is `laravel`.
3. `--test` means: do not change anything, only report. It exits with code **0** if everything is fine
   and **1** if any file would change. CI uses that exit code to decide red or green.

**Check it works:** if you copied the code from lesson 01 exactly, you see `PASS` and something like
`37 files`. If you typed code yourself, you may see `FAIL` with a list of files and the names of the
rules they break (for example `braces_position`, `concat_space`, `no_unused_imports`).

Now let Pint fix everything:

```bash
./vendor/bin/pint
./vendor/bin/pint --test
```

Without `--test`, Pint **rewrites** the files. The second command must now show `PASS`.

Look at what changed:

```bash
git diff
```

**Commit** (only if Pint changed something) — as a **separate** commit, so formatting never hides a
real change:

```bash
git add -A
git commit -m "style: apply Pint to existing code"
```

---

## Step 3 — `pint.json`

**Why:** without a configuration file, Pint silently uses the `laravel` preset. Writing it down makes
the decision **visible**: anyone opening the repository sees which standard applies, and `CODESTYLE.md`
can point to the file.

Create `pint.json` in the project root:

```json
{
    "preset": "laravel"
}
```

(JSON files cannot contain comments, so this file has no path comment. It lives at `pint.json`, next to
`composer.json`.)

**What happens:**

1. `"preset": "laravel"` selects Laravel's rule set, based on PSR-12 / PER Coding Style plus Laravel's
   own habits. Other presets exist (`psr12`, `per`, `symfony`), but the `laravel` preset is what Laravel
   itself and most Laravel projects use.
2. You *could* add a `"rules"` object to switch individual rules on or off. We do not. Every custom rule
   is a decision the whole group would have to make and maintain. If the group ever changes a rule, it
   is recorded in `CODESTYLE.md` with a date.

**Blade files** are not formatted by Pint — it only understands PHP. For views we rely on
`.editorconfig` (next step) and on reviews.

**Check it works:** `./vendor/bin/pint --test` still shows `PASS`. Add `-v` to see which rules were
applied: `./vendor/bin/pint --test -v`.

**Commit:**

```bash
git add pint.json
git commit -m "chore: add Pint configuration with laravel preset"
```

---

## Step 4 — `.editorconfig`

**Why:** Pint runs when you ask. Your editor changes the file every time you type. `.editorconfig`
makes every editor in the group use the same encoding, line endings and indentation — before Pint
even sees the code.

Laravel already ships an `.editorconfig`. Open it, then replace its content with this version (the
difference: two-space indentation for **all** YAML files, including our `compose.yaml` and the GitHub
workflow we write in Step 6):

```ini
# .editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
indent_size = 4
indent_style = space
insert_final_newline = true
trim_trailing_whitespace = true

[*.md]
trim_trailing_whitespace = false

[*.{yml,yaml}]
indent_size = 2
```

**What happens:**

1. `root = true` — editors stop looking for more `.editorconfig` files in parent folders.
2. `[*]` applies to every file: UTF-8, **LF** line endings, 4-space indentation (PHP, Blade, JSON,
   JS), a newline at the end of the file, no spaces at the end of lines.
3. `[*.md]` — in Markdown, two spaces at the end of a line mean "line break", so we must not remove them.
4. `[*.{yml,yaml}]` — YAML is usually indented with 2 spaces.

**Editor support:** PhpStorm and IntelliJ read `.editorconfig` automatically. **VS Code** needs the
extension *EditorConfig for VS Code* — install it now.

**Windows users:** Git can also convert line endings. Check your setting with
`git config --get core.autocrlf`. Laravel ships a `.gitattributes` file with `* text=auto eol=lf`, which
tells Git to store and check out text files with LF regardless of that setting. Leave that file as it is.

**Check it works:** open any PHP file, press Enter at the end of a line and look at the indentation and
the line-ending indicator in the status bar of your editor (`LF`).

**Commit:**

```bash
git add .editorconfig
git commit -m "chore: align editorconfig with project conventions"
```

---

## Step 5 — `CODESTYLE.md` and the group vote

**Why:** this file is the **agreement** that ÕV7 talks about. The tools enforce layout; this file
records everything else, and when and how the group agreed on it.

**In class:** the teacher shows the three discretionary items from the
[concepts page](README.md#11-codestylemd--writing-the-agreement-down). The group votes by show of hands.
Everyone writes the **same result** and the **same date** into their own `CODESTYLE.md`. (If the group
chooses the other option for an item, change the text accordingly — the template below shows one
possible result.)

Create `CODESTYLE.md` in the project root:

```markdown
<!-- CODESTYLE.md -->
# Code standard — Billable

**Agreed by the group on 2026-10-07** (lesson 02). Changes need a group decision and a new date
in the change log at the end of this file.

## 1. Tools

| What | Tool | Command |
|---|---|---|
| Format all PHP files | Laravel Pint, preset `laravel` (`pint.json`) | `./vendor/bin/pint` |
| Check formatting (CI does this) | Laravel Pint | `./vendor/bin/pint --test` |
| Editor settings | `.editorconfig` | automatic (VS Code: install the EditorConfig extension) |

Run `./vendor/bin/pint` before every commit. A pull request with a red CI check is not reviewed.

The formatter is the authority on layout. We do not discuss formatting in reviews.

## 2. Language

All identifiers, comments, commit messages and documentation are in **English**. We use the domain
words from PROJECT.md: client, project, time entry, tag, invoice, invoice line.

## 3. Naming

| Thing | Convention | Example |
|---|---|---|
| Class, interface, enum | UpperCamelCase noun | `InvoiceGenerator`, `BillingType` |
| Method | camelCase verb | `calculateVat()` |
| Boolean method | question | `isDraft()`, `hasBudgetCap()` |
| Variable, property | camelCase, with unit where relevant | `$amountCents`, `$roundedMinutes` |
| Constant | UPPER_SNAKE_CASE | `VAT_RATE_BP` |
| Enum case | UpperCamelCase | `BillingType::Hourly` |
| Table | plural snake_case | `time_entries` |
| Column | snake_case | `hourly_rate_cents` |
| Route name | `resource.action` | `clients.index` |
| URL | lowercase, plural, hyphens | `/time-entries` |
| View | folder per resource | `resources/views/clients/index.blade.php` |
| Test method | **snake_case sentence with `test_` prefix** (vote) | `test_capped_billing_never_exceeds_cap` |

## 4. Rules

- Controllers stay thin: no calculations, no business rules. Views contain no queries.
- Money is always integer cents (`int`). Never `float` for money.
- No `dd()`, `dump()`, `var_dump()` or commented-out code in committed code.
- Type declarations on all parameters, return types and properties.
- Doc comments: **required on public methods of domain classes** (`app/Billing`, `app/Support`,
  `app/Invoicing`, `app/Services`); optional elsewhere (vote).

## 5. Commits

Conventional Commits: `type(scope): summary` in imperative mood, first line under 72 characters.
Types: `feat`, `fix`, `refactor`, `test`, `docs`, `style`, `chore`, `ci`.
Scopes: `home`, `clients`, `projects`, `time-entries`, `billing`, `invoices`, `reports`.
Formatting-only changes go in their own `style:` commit.

## 6. Branches, pull requests and review

- No direct commits to `main`. Every change goes through a branch and a pull request.
- Branch names: `type/short-description`, e.g. `feature/client-list`, `fix/vat-rounding`.
- A pull request needs a **green CI check** and **one approving review from a classmate** before merge.
- Merge method: **Squash and merge** (vote). The squash commit message follows section 5.
- Reviews comment on correctness, names, structure, tests, security and docs — not on formatting.
- The author replies to every review comment.

## Change log

| Date | Change |
|---|---|
| 2026-10-07 | First version agreed by the group. |
```

Replace `2026-10-07` with the **actual date of your lesson 02**, in both places.

**What happens:**

1. The date at the top is evidence that the standard existed **before** most of the code. Your Git
   history will confirm it: the commit that adds this file comes before the commits for clients,
   projects and invoices.
2. The file points to the tools instead of repeating their rules. Nobody needs a list of 200 spacing
   rules — Pint has them.
3. The rules in section 4 are things **no formatter can check**. Each one is tied to Billable:
   thin controllers (ÕV3), integer cents (R1), no debug calls.
4. The vote results are marked "(vote)", so a reader knows which rules were a free choice.

**Commit:**

```bash
git add CODESTYLE.md
git commit -m "docs: add CODESTYLE.md agreed by the group"
```

---

## Step 6 — CI with GitHub Actions

**Why:** a standard that is not checked automatically is a suggestion. The CI workflow makes it a rule:
every push and every pull request is checked on a clean machine.

Create the folders and the file `.github/workflows/ci.yml`:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  code-style:
    name: Code style (Pint)
    runs-on: ubuntu-latest

    steps:
      - name: Check out code
        uses: actions/checkout@v5

      - name: Set up PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'
          tools: composer:v2
          coverage: none

      - name: Install Composer dependencies
        run: composer install --no-interaction --prefer-dist --no-progress

      - name: Check code style
        run: ./vendor/bin/pint --test
```

**What happens:**

1. `on:` — the workflow runs on every push to `main` and on every pull request (any branch, every new
   push to the PR). Pushes to a feature branch without a PR do not run it; the PR does.
2. `jobs.code-style` — one **job** that runs on a fresh Ubuntu virtual machine provided by GitHub.
   Nothing from your laptop is there — only what is committed to Git.
3. `actions/checkout@v5` downloads your repository into the machine.
4. `shivammathur/setup-php@v2` installs PHP 8.4 and Composer. `coverage: none` skips the code coverage
   extension, which we do not need here (and which makes PHP slower).
5. `composer install` installs exactly the versions in `composer.lock`, including Pint. `vendor/` is
   not in Git, so CI must install it every time. `--no-interaction` because nobody can answer
   questions on a CI machine.
6. `./vendor/bin/pint --test` — the actual check. Exit code 1 → the step fails → the job fails → the
   check on the pull request turns **red**.

**Tests are not in CI yet.** We have no tests of our own yet. In lesson 11 we add a second job to this
same file that runs `php artisan test`. Then "green" means "formatted **and** tested".

**Commit:**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: check code style with Pint on push and pull request"
```

---

## Step 7 — Push the branch and open a pull request

```bash
git push -u origin chore/code-standard
```

**What happens:** `-u` (upstream) connects your local branch with the new branch on GitHub, so later
you can type just `git push`.

Now open the pull request:

1. Go to your repository on GitHub. A yellow banner says *"chore/code-standard had recent pushes"* →
   click **Compare & pull request**.
2. Base: `main`, compare: `chore/code-standard`.
3. Title: `chore: add agreed code standard and CI style check`.
4. Description — use this structure for every PR from now on:

```markdown
## What
- Pint configuration (`laravel` preset) and formatting of existing code
- `.editorconfig`
- `CODESTYLE.md` as agreed in lesson 02
- GitHub Actions workflow that runs `pint --test`

## Why
Lesson 02: the group agreed on a code standard; CI enforces it.

## How to test
Run `./vendor/bin/pint --test` — it should print PASS.
```

5. Click **Create pull request**.

**Check it works:** at the bottom of the PR page you see **"CI / Code style (Pint)"** with a yellow dot
(running), and after about a minute a green check mark. Click **Details** to see the log of every step.
The **Actions** tab of the repository lists the same run.

---

## Step 8 — Break it on purpose: red, then green

**Why:** you need to see that the gate actually closes. A CI check that has never been red has not been
tested.

Make a formatting mess in `app/Http/Controllers/HomeController.php` — change the `index` method to:

```php
// app/Http/Controllers/HomeController.php — deliberately badly formatted (temporary)
    public function index() : View {
        return view('home',['today'=>today()]);
    }
```

The code still works (reload the page — it looks the same). Only the formatting is wrong. Commit and
push **without** running Pint:

```bash
git commit -am "chore: break formatting on purpose to test CI"
git push
```

**Check it works:** on the PR page, the check turns **red** (a red ✗). Click **Details**. In the
"Check code style" step, Pint's output looks like this:

```
  ⨯

  FAIL   ...................................... 38 files, 1 style issue
  ⨯ app/Http/Controllers/HomeController.php      braces_position, binary_operator_spaces, …
Error: Process completed with exit code 1.
```

The exact list of rule names may differ. Note that the reviewer did not have to notice anything —
the machine did.

Now fix it the normal way:

```bash
./vendor/bin/pint
git diff
git commit -am "style: fix formatting in HomeController"
git push
```

**Check it works:** `git diff` shows Pint restored the original layout. On GitHub, the new run turns
**green**. The PR's commit list now shows the red ✗ next to the "break formatting" commit and the green
✓ next to the fix. This red-then-green pair is evidence for ÕV7 — **leave both commits in the history**
(if your group voted for "squash and merge", the individual commits remain visible in the closed pull
request).

---

## Step 9 — Review and merge

In class, swap with your neighbour: open their PR (they add you as a collaborator — see Task 1), read
the **Files changed** tab and leave one comment that is *not* about formatting. A good first comment on
this PR might be about `CODESTYLE.md`: "Section 4 says *no business logic in controllers* — should we
add an example of what counts as business logic?"

Then merge your own PR:

1. Click **Squash and merge** (or **Create a merge commit** — whichever your group voted for).
2. Check that the commit message follows Conventional Commits:
   `chore: add agreed code standard and CI style check (#1)`.
3. **Confirm**, then **Delete branch**.

Update your laptop:

```bash
git switch main
git pull
git branch -d chore/code-standard
```

**Check it works:** `git log --oneline -5` shows the merged work on `main`. The Actions tab shows a
new run for the push to `main` — also green.

---

## Step 10 — Protect `main` (if available)

**Why:** so far the red check only *warns*. Branch protection makes GitHub refuse the merge button
while CI is red.

On GitHub: **Settings → Branches → Add branch protection rule** (newer interface: **Settings → Rules →
Rulesets → New branch ruleset**).

- Branch name pattern / target: `main`
- Require a pull request before merging
- Require status checks to pass → select **Code style (Pint)** (it appears in the list after it has
  run at least once)

Save. **Note:** on a *private* repository with a free GitHub account, these rules may not be enforced.
They work on public repositories and with GitHub Pro, which is free for students through the GitHub
Student Developer Pack. If you cannot enable it, the rule "no merge while red" is still in your
`CODESTYLE.md`, and you follow it by hand.

---

## Troubleshooting

| Error / symptom | Cause | Fix |
|---|---|---|
| `./vendor/bin/pint: No such file or directory` | Dependencies not installed, or you are not in the project root | `composer install`; run the command from the folder that contains `artisan`. |
| `'.' is not recognized as an internal or external command` (Windows) | PowerShell / cmd syntax | Use `vendor\bin\pint` or `php vendor/bin/pint`. |
| Pint changes files you did not touch | Those files were never formatted (e.g. copied code) | Fine — commit them in a separate `style:` commit. |
| CI fails at `composer install` with `Your lock file does not contain a compatible set of packages` / PHP version error | `composer.lock` needs a different PHP version than CI uses | Make sure `php-version: '8.4'` matches your local PHP (`php -v`). Commit `composer.lock`. |
| CI fails with `Could not open input file: artisan` during `composer install` | The workflow runs in the wrong folder (project is in a sub-folder of the repository) | Keep the Laravel project at the repository root, or add `working-directory:` to the steps. |
| The workflow does not run at all | File in the wrong place or wrong extension | Must be `.github/workflows/ci.yml` (note the dot in `.github`). Check the YAML indentation (2 spaces, no tabs). |
| `pint --test` passes locally, fails in CI | A file is formatted locally but the formatted version was not committed | `git status` — commit the changes; run `./vendor/bin/pint --test` after `git stash` to be sure. |
| Every line shows as changed in `git diff` | Line endings were converted (CRLF ↔ LF) | Keep Laravel's `.gitattributes`; set the editor to LF; `git add --renormalize .` once and commit as `style: normalise line endings`. |
| Push rejected: `Updates were rejected because the remote contains work that you do not have` | Someone (or you, on GitHub) changed the branch | `git pull --rebase`, then `git push`. |
| The status check does not appear in branch protection | It has never run on the repository | Push once so the workflow runs, then search for the job name `Code style (Pint)`. |

---

## Recap

- `pint.json` — the `laravel` preset, written down so the choice is visible.
- `.editorconfig` — UTF-8, LF, 4 spaces (2 for YAML), final newline, for every editor.
- `CODESTYLE.md` — the agreement: date, tools and commands, language, naming, rules, commits, review
  process, and the vote results.
- `.github/workflows/ci.yml` — installs dependencies and runs `pint --test` on pushes to `main` and on
  every pull request. Tests join it in lesson 11.
- Branch `chore/code-standard` → pull request → red (deliberate) → green → reviewed → merged.
- Formatting commits are separate (`style:`), and nobody comments on formatting in reviews.

---

## Independent work — solutions

<details>
<summary>Task 1 — A reviewed pull request (Basic)</summary>

**Give your reviewer access.** On a private repository your classmate needs access: **Settings →
Collaborators → Add people**. Then on the PR page, click the gear next to **Reviewers** and select them.

**Example PR:** branch `feature/about-nav`, implementing lesson 01 Task 3 (named routes and active nav
link). Title: `feat(home): highlight active page in navigation`.

**Examples of useful, non-formatting review comments** on such a PR:

1. On `layouts/app.blade.php`: "The `@if (request()->routeIs(...))` expression is repeated for every
   link. When we add Clients and Projects, this line gets copied four more times. Could a small Blade
   component or a helper take the route name and render the link?"
2. On `routes/web.php`: "`routeIs('home')` will not match sub-pages. When we add `clients.show`, should
   `clients.*` also mark *Clients* as active? `routeIs` accepts wildcards: `routeIs('clients.*')`."
3. On `about.blade.php`: "The text says 'tracks your hours' but Billable also invoices — maybe mention
   invoices, since that is the main feature?"
4. On the PR description: "How do I test this? Could you add the steps (visit `/`, then `/about`, check
   which link is highlighted)?"

**Replying:** for each comment, either push a commit and reply "Done in `3f2a1bc`", or explain: "I would
rather wait with the component until lesson 04, when we have more links — I added a note to the README
TODO list." Then click **Resolve conversation**.

**Merge** with the method from `CODESTYLE.md`, delete the branch, and `git switch main && git pull`.

</details>

<details>
<summary>Task 2 — Clean commit history habits (Basic)</summary>

A practice run on a branch:

```bash
git switch main && git pull
git switch -c docs/readme-how-to-run
```

**Partial staging with `git add -p`.** Change two unrelated things in `README.md` (a "How to run"
section and a typo elsewhere). Then:

```bash
git add -p README.md
```

Git shows each changed block ("hunk") and asks `Stage this hunk [y,n,q,a,d,s,e,?]?`. Press `y` for the
"How to run" hunk and `n` for the typo. Commit:

```bash
git commit -m "docs: add how to run section to readme"
git add README.md
git commit -m "docs: fix typo in readme intro"
```

**Fix the last message with `--amend`.** Suppose the second message should have been different:

```bash
git commit --amend -m "docs: fix typo in project description"
```

`--amend` replaces the **last** commit. Only do this before you push (after pushing you would have to
force-push, which rewrites shared history).

**Combine a fix-up commit.** You notice a mistake in the "How to run" section:

```bash
# edit README.md, then:
git add README.md
git commit --fixup=HEAD~1
git rebase -i --autosquash main
```

`--fixup=HEAD~1` creates a commit called `fixup! docs: add how to run section to readme`. The
interactive rebase opens an editor with the list of commits, and `--autosquash` has already moved the
fix-up commit under its target and marked it `fixup`. Save and close the editor. The fix-up is now
part of the original commit.

(Without `--fixup`: run `git rebase -i main`, change `pick` to `fixup` — or `f` — in front of the commit
you want to merge into the line above it, save and close.)

**Check:**

```bash
git log --oneline main..HEAD
```

shows exactly two clean commits. Push and open the PR.

**Rule to remember:** rewrite history (`--amend`, `rebase`) only on **your own branch, before others
build on it**. If you already pushed the branch, use `git push --force-with-lease` (it refuses to
overwrite work you have not seen). Never rewrite `main`.

**Add to `CODESTYLE.md`** (your own words):

```markdown
<!-- CODESTYLE.md — add after section 5 -->
### Commit habits
- One logical change per commit; use `git add -p` to split unrelated changes.
- Read `git diff --staged` before every commit.
- Tidy up with `--amend` and `rebase -i --autosquash` before opening a PR — never on `main`.
- After rewriting a pushed branch, push with `--force-with-lease`, never plain `--force`.
```

</details>

<details>
<summary>Task 3 — A shared pre-commit hook (Intermediate)</summary>

Git looks for hooks in `.git/hooks/` by default, and that folder is **not** part of the repository.
We put the hook in a normal, committed folder and tell Git to look there.

```sh
#!/bin/sh
# hooks/pre-commit
# Refuses the commit if Pint finds formatting problems.
# Activate once per clone:  git config core.hooksPath hooks

if ! ./vendor/bin/pint --test --dirty > /dev/null 2>&1; then
  echo ""
  echo "✗ Code style check failed."
  echo "  Run:  ./vendor/bin/pint"
  echo "  Then add the changes and commit again."
  echo ""
  exit 1
fi

exit 0
```

Make it executable and activate it:

```bash
chmod +x hooks/pre-commit
git config core.hooksPath hooks
git add hooks/pre-commit
git update-index --chmod=+x hooks/pre-commit
git commit -m "chore: add shared pre-commit hook for Pint"
```

**What happens:**

1. Git runs `hooks/pre-commit` before creating each commit. A non-zero exit code aborts the commit.
2. `pint --test --dirty` checks only the files that Git sees as changed (staged, modified or new), so
   the hook stays fast in a big project. CI still checks the whole project with `pint --test`. The
   output is hidden (`> /dev/null 2>&1`) and replaced with a short, clear message.
3. `core.hooksPath` is a **local** setting (stored in `.git/config`), so every teammate must run the
   `git config` command once after cloning. That is why it goes into `CODESTYLE.md`.
4. `git update-index --chmod=+x` stores the executable bit in Git, important for Windows users.
   Git for Windows runs `sh` hooks with its bundled shell.

**Test it:** break the formatting in `HomeController.php` and run `git commit -am "test"`. The commit is
refused with the message above. Run Pint, commit again — it works.

**Add to `CODESTYLE.md`** (section 1):

```markdown
<!-- CODESTYLE.md — add to section 1 -->
**Pre-commit hook (optional, recommended):** run `git config core.hooksPath hooks` once after cloning.
The hook only gives faster feedback: it runs on your machine, can be skipped with `--no-verify`, and is
not installed automatically. CI is the gate that nothing can skip.
```

</details>

<details>
<summary>Task 4 — Larastan in CI (Advanced)</summary>

Install Larastan (the Laravel extension for PHPStan) as a development dependency:

```bash
composer require --dev larastan/larastan
```

Create `phpstan.neon` in the project root:

```neon
# phpstan.neon
includes:
    - vendor/larastan/larastan/extension.neon

parameters:
    paths:
        - app/
    level: 5
```

Run it:

```bash
./vendor/bin/phpstan analyse
```

**What happens:**

1. PHPStan reads your code **without running it** and builds a model of all types. Larastan teaches it
   Laravel's "magic" (facades, Eloquent models, `config()` return types), which plain PHPStan would
   report as errors.
2. `level: 5` — PHPStan has levels 0 (basic checks: unknown classes, wrong number of arguments) to 10
   (strictest). Level 5 adds checks of argument types passed to methods and functions. It is a good
   start: strict enough to find real bugs, not so strict that a junior spends the evening on it.
3. `paths: app/` — we analyse our own code, not `vendor/` or `tests/`.

**Check it works:** the output ends with `[OK] No errors`. If it runs out of memory, add
`--memory-limit=1G`.

Try a deliberate error to see it work — in `HomeController::index()` temporarily write
`return view('home', ['today' => $this->todayInTallinn()]);`, a method that does not exist. PHPStan
reports `Call to an undefined method App\Http\Controllers\HomeController::todayInTallinn()`. Pint would
not notice this at all, and PHP itself only notices when someone opens the page.

**Add it to CI** — in `.github/workflows/ci.yml`, add a step after "Check code style":

```yaml
# .github/workflows/ci.yml — add as the last step of the code-style job
      - name: Static analysis
        run: ./vendor/bin/phpstan analyse --no-progress --error-format=github
```

`--error-format=github` makes errors appear as annotations directly on the lines in the PR's
**Files changed** tab. You may also want to rename the job to `Code quality`; remember to update the
required status check name in branch protection if you do.

**Add to `CODESTYLE.md`** (section 1, tools table):

```markdown
| Static analysis (CI does this) | Larastan, PHPStan level 5 (`phpstan.neon`) | `./vendor/bin/phpstan analyse` |
```

```bash
git add composer.json composer.lock phpstan.neon .github/workflows/ci.yml CODESTYLE.md
git commit -m "ci: add Larastan static analysis at level 5"
```

</details>
