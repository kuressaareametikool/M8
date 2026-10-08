# 02 — An agreed code standard · Spring Boot

[← Concepts](README.md) · Starting point: end of lesson 01 · Estimated time in class: 2 h · Independent work tasks: [README](README.md#independent-work-2-h)

---

## What we build today

- The **Spotless** Maven plugin with **google-java-format**, bound to the `validate` phase so every
  build checks formatting. The whole project formatted once with `./mvnw spotless:apply`.
- An **`.editorconfig`** for Java, HTML, YAML and the POM.
- **`CODESTYLE.md`** — the group's agreement, with the date and the results of the vote.
- A **GitHub Actions** workflow that runs `./mvnw -B verify` (formatting check + the existing test) with
  Java 25 (Temurin). The test brings its own PostgreSQL through Testcontainers (lesson 01).
- A **pull request** `chore/code-standard` that we deliberately break (CI red), fix (CI green) and
  merge.

At the end, the **Actions** tab of your GitHub repository shows a red run and a green run, and `main`
contains all of the above.

We work on a **branch** from the start today — this lesson is where we stop committing directly to
`main`.

---

## Step 1 — Start a branch

**Why:** everything we add today reaches `main` through a pull request, so you see the whole team
workflow once, end to end.

```bash
git switch main
git pull
git status
git switch -c chore/code-standard
```

**What happens:**

1. `git pull` brings your local `main` up to date with GitHub.
2. `git status` must say `nothing to commit, working tree clean`. If not, commit or stash first.
3. `git switch -c chore/code-standard` creates the branch and switches to it. The name follows the
   pattern `type/short-description`, with the same types as commit messages.

**Check it works:** `git branch` shows `* chore/code-standard`.

---

## Step 2 — Add Spotless to `pom.xml`

**Why:** google-java-format implements the Google Java Style Guide exactly and cannot be configured —
so there is nothing to argue about. Spotless runs it inside Maven, which means the **build itself**
checks formatting: on your laptop, in IntelliJ and in CI, with the same version.

**Find the current plugin version.** Open Maven Central and search for
`com.diffplug.spotless:spotless-maven-plugin`
(https://central.sonatype.com/artifact/com.diffplug.spotless/spotless-maven-plugin). Note the newest
version number. **Do not copy a version from an old tutorial:** older Spotless versions bundle an old
google-java-format that crashes on new JDKs (last year's group hit
`NoSuchMethodError: com.sun.tools.javac.util.Log$DeferredDiagnosticHandler.getDiagnostics()` on a new
Java). With Java 25 you need a recent version.

Open `pom.xml`. In the `<properties>` section, next to `<java.version>`, add the version property:

```xml
<!-- pom.xml — inside <properties> -->
<properties>
	<java.version>25</java.version>
	<spotless.version>REPLACE_WITH_LATEST_VERSION</spotless.version>
</properties>
```

Replace `REPLACE_WITH_LATEST_VERSION` with the exact version number that Maven Central shows as the
latest release.

Then find `<build>` → `<plugins>`. It already contains `spring-boot-maven-plugin`. Add the Spotless
plugin **after** it:

```xml
<!-- pom.xml — inside <build><plugins>, after spring-boot-maven-plugin -->
<plugin>
	<groupId>com.diffplug.spotless</groupId>
	<artifactId>spotless-maven-plugin</artifactId>
	<version>${spotless.version}</version>
	<configuration>
		<java>
			<googleJavaFormat/>
			<removeUnusedImports/>
		</java>
	</configuration>
	<executions>
		<execution>
			<goals>
				<goal>check</goal>
			</goals>
			<phase>validate</phase>
		</execution>
	</executions>
</plugin>
```

(Initializr's `pom.xml` is indented with **tabs**. Keep using tabs in this file, so the diff only shows
the lines you added.)

**What happens:**

1. `<version>${spotless.version}</version>` — Spring Boot's parent POM does **not** manage Spotless's
   version, so we must give one. Keeping it in a property makes updates a one-line change.
2. `<java>` — configure formatting for Java source files in `src/main/java` and `src/test/java`.
3. `<googleJavaFormat/>` — use google-java-format. Without a `<version>` inside, Spotless uses the
   google-java-format version it was released with; a recent Spotless means a recent formatter.
4. `<removeUnusedImports/>` — delete imports nobody uses.
5. `<executions>` — run the goal `check` in the phase `validate`. Maven's build **lifecycle** runs phases
   in order: `validate` → `compile` → `test` → `package` → `verify` → … Because `validate` is the
   **first** phase, `./mvnw compile`, `./mvnw test`, `./mvnw verify` and `./mvnw package` all check
   formatting first, and stop if it is wrong.

---

## Step 3 — Check, then apply

Run the check alone:

```bash
./mvnw spotless:check
```

(Windows PowerShell: `.\mvnw spotless:check`.)

**Check it works:** it **fails**, and that is expected. Initializr generates Java files indented with
tabs, and google-java-format uses 2 spaces. The output looks like this:

```
[ERROR] Failed to execute goal com.diffplug.spotless:spotless-maven-plugin:…:check (default-cli)
  on project billable: The following files had format violations:
[ERROR]     src/main/java/ee/ta25/billable/BillableApplication.java
[ERROR]         @@ -6,8 +6,8 @@
[ERROR]          public class BillableApplication {
[ERROR]         -→public static void main(String[] args) {
[ERROR]         +··public static void main(String[] args) {
…
[ERROR] Run 'mvn spotless:apply' to fix these violations.
```

`→` is a tab and `·` is a space. `HomeController.java` from lesson 01 was already written in Google
style, so it may not be listed.

Now let Spotless fix everything:

```bash
./mvnw spotless:apply
./mvnw spotless:check
git diff --stat
```

**What happens:**

1. `spotless:apply` **rewrites** the files that break the format.
2. The second `spotless:check` prints `BUILD SUCCESS`.
3. `git diff --stat` lists the changed files — normally `BillableApplication.java`,
   `BillableApplicationTests.java` and, if you edited the generated file instead of retyping it,
   `TestcontainersConfiguration.java`.

Now see that the normal build checks formatting too:

```bash
./mvnw verify
```

(Docker Desktop must be running: the `contextLoads` test starts its own PostgreSQL container, as in
lesson 01.)

The log shows `spotless-maven-plugin:…:check (default)` **before** `maven-compiler-plugin`, and ends with
`BUILD SUCCESS`.

**Commit** — the plugin and the formatting in **separate** commits, so the formatting never hides a real
change:

```bash
git add pom.xml
git commit -m "chore: add Spotless with google-java-format bound to validate"

git add -A
git commit -m "style: apply google-java-format to generated code"
```

**IntelliJ tip:** IntelliJ's own "Reformat Code" (Ctrl+Alt+L / ⌥⌘L) does not produce exactly the same
result as google-java-format. Either install the **google-java-format** plugin from the JetBrains
Marketplace and enable it in *Settings → google-java-format Settings*, or simply run
`./mvnw spotless:apply` before committing. The Maven command is the authority.

---

## Step 4 — `.editorconfig` and line endings

**Why:** Spotless runs when you ask. Your editor changes files every time you type. `.editorconfig`
makes every editor in the group use the same encoding, line endings and indentation from the start.

Create `.editorconfig` in the project root:

```ini
# .editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 4
insert_final_newline = true
trim_trailing_whitespace = true

[*.java]
indent_size = 2
max_line_length = 100

[*.{html,yml,yaml,json}]
indent_size = 2

[pom.xml]
indent_style = tab

[*.md]
trim_trailing_whitespace = false

[*.cmd]
end_of_line = crlf
```

**What happens:**

1. `[*]` — defaults for every file: UTF-8, **LF** line endings, spaces, final newline, no trailing
   spaces.
2. `[*.java]` — 2 spaces and 100 columns, the same as google-java-format, so your editor and the
   formatter agree while you type.
3. `[*.{html,…}]` — our Thymeleaf templates, `compose.yaml` and the workflow use 2 spaces.
4. `[pom.xml]` — tabs, because that is how Initializr generated it. Converting the whole POM to spaces
   would create a big meaningless diff.
5. `[*.md]` — two trailing spaces mean "line break" in Markdown; do not remove them.
6. `[*.cmd]` — `mvnw.cmd` is a Windows script and needs Windows (CRLF) line endings.

IntelliJ reads `.editorconfig` automatically. **VS Code** needs the extension *EditorConfig for VS
Code*.

**Line endings in Git.** Initializr created a `.gitattributes` with lines for `mvnw` and `*.cmd`. Add
one line **at the top**, so every other text file is stored with LF no matter what the Windows setting
`core.autocrlf` is:

```gitattributes
# .gitattributes — add as the first line, keep the generated lines below it
* text=auto eol=lf
```

Later lines override earlier ones, so `*.cmd text eol=crlf` further down still wins for `mvnw.cmd`.

**Commit:**

```bash
git add .editorconfig .gitattributes
git commit -m "chore: add editorconfig and normalise line endings"
```

---

## Step 5 — `CODESTYLE.md` and the group vote

**Why:** this file is the **agreement** that ÕV7 is about. The formatter enforces layout; this file
records everything else, and when and how the group agreed on it.

**In class:** the teacher shows the three discretionary items from the
[concepts page](README.md#11-codestylemd--writing-the-agreement-down). The group votes by show of hands.
Everyone writes the **same result** and the **same date** into their own `CODESTYLE.md`. (If the group
chose differently, change the text — the template shows one possible result.)

Create `CODESTYLE.md` in the project root:

```markdown
<!-- CODESTYLE.md -->
# Code standard — Billable

**Agreed by the group on 2026-10-07** (lesson 02). Changes need a group decision and a new date
in the change log at the end of this file.

## 1. Tools

| What | Tool | Command |
|---|---|---|
| Format all Java files | Spotless + google-java-format (`pom.xml`) | `./mvnw spotless:apply` |
| Check formatting | Spotless, bound to `validate` — every build checks it | `./mvnw spotless:check` |
| Full check (CI does this) | Maven | `./mvnw verify` |
| Editor settings | `.editorconfig` | automatic (VS Code: install the EditorConfig extension) |

Run `./mvnw spotless:apply` before every commit. A pull request with a red CI check is not reviewed.

The formatter is the authority on layout. We do not discuss formatting in reviews.

## 2. Language

All identifiers, comments, commit messages and documentation are in **English**. We use the domain
words from PROJECT.md: client, project, time entry, tag, invoice, invoice line.

## 3. Naming

| Thing | Convention | Example |
|---|---|---|
| Class, interface, record, enum | UpperCamelCase noun | `InvoiceGenerator`, `BillingType` |
| Method | lowerCamelCase verb | `calculateVat()` |
| Boolean method | question | `isDraft()`, `hasBudgetCap()` |
| Variable, field | lowerCamelCase, with unit where relevant | `amountCents`, `roundedMinutes` |
| Constant | UPPER_SNAKE_CASE | `VAT_RATE_BP` |
| Enum constant | UPPER_SNAKE_CASE | `BillingType.HOURLY` |
| Package | lowercase, singular, by feature | `ee.ta25.billable.timeentry` |
| Table | plural snake_case | `time_entries` |
| Column | snake_case | `hourly_rate_cents` |
| Migration | `V<n>__<description>.sql` | `V2__create_projects.sql` |
| URL | lowercase, plural, hyphens | `/time-entries` |
| Template | folder per feature | `templates/clients/index.html` |
| Test method | **lowerCamelCase describing the behaviour** (vote) | `cappedBillingNeverExceedsCap()` |

## 4. Rules

- Controllers stay thin: no calculations, no business rules. Templates contain no logic beyond
  display conditions.
- Money is always integer cents (`long`). Never `float` or `double` for money.
- Constructor injection only. No `@Autowired` on fields.
- No `System.out.println`, `printStackTrace()` or commented-out code in committed code. Use a logger.
- `jakarta.*` imports only (never `javax.persistence` / `javax.validation`).
- Javadoc: **required on public methods of domain classes** (`billing`, `invoice`, `common` value
  objects, services); optional elsewhere (vote).

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

1. The date shows the standard existed **before** most of the code. Your Git history confirms it: the
   commit adding this file comes before the commits for clients, projects and invoices.
2. The file points to the tools instead of repeating their rules — google-java-format has hundreds of
   rules, and nobody needs them in a Markdown file.
3. Section 4 lists what **no formatter can check**, each tied to Billable: thin controllers (ÕV3),
   integer cents (R1), constructor injection (lesson 07), Boot 3+ `jakarta` packages.
4. The vote results are marked "(vote)", so a reader knows which rules were a free choice.

**Commit:**

```bash
git add CODESTYLE.md
git commit -m "docs: add CODESTYLE.md agreed by the group"
```

---

## Step 6 — CI with GitHub Actions

**Why:** a standard that is not checked automatically is a suggestion. The workflow makes it a rule:
every push and pull request is built on a clean machine.

Create `.github/workflows/ci.yml`:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    name: Build and verify
    runs-on: ubuntu-latest

    steps:
      - name: Check out code
        uses: actions/checkout@v5

      - name: Set up Java 25
        uses: actions/setup-java@v5
        with:
          distribution: temurin
          java-version: '25'
          cache: maven

      - name: Build, check formatting and run tests
        run: ./mvnw -B verify
```

**What happens:**

1. `on:` — the workflow runs on every push to `main` and on every pull request (and every new push to
   an open PR).
2. `runs-on: ubuntu-latest` — a fresh Linux virtual machine. Only what is committed to Git is there.
3. **No database service.** `./mvnw verify` also runs the tests, and the `contextLoads` test starts the
   whole application, including the database connection and Flyway (lesson 01, Step 9). It gets its
   database from `TestcontainersConfiguration`, exactly as on your laptop. GitHub's `ubuntu-latest`
   runners have Docker installed and running, so Testcontainers can start `postgres:17-alpine` there
   too. Many CI tutorials add a `services: postgres` block to the job instead; we do not need one,
   because the test does not connect to `localhost:5432` at all. One database setup for laptop and CI
   means one thing less that can differ.
4. `actions/setup-java@v5` with `distribution: temurin` and `java-version: '25'` installs Eclipse
   Temurin JDK 25. `cache: maven` stores downloaded dependencies between runs, so later runs are much
   faster.
5. `./mvnw -B verify` — `-B` is **batch mode**: no colours and no download progress bars in the log.
   `verify` runs the lifecycle up to `verify`: **Spotless check (validate)** → compile → tests →
   package. If formatting is wrong, the build stops at the very first phase and the check is red.

In lesson 11 we write many more tests, and in lesson 14 integration tests; they all run in this same
step automatically, on the same Testcontainers database.

**Windows users:** CI runs `./mvnw` on Linux, so the file must be executable in Git. If you did not do
it in lesson 01:

```bash
git update-index --chmod=+x mvnw
```

**Commit:**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: run maven verify with java 25 on push and pull request"
```

---

## Step 7 — Push the branch and open a pull request

```bash
git push -u origin chore/code-standard
```

`-u` connects your local branch with the new branch on GitHub, so later a plain `git push` is enough.

Open the pull request:

1. On your repository page, click **Compare & pull request** in the yellow banner.
2. Base: `main`, compare: `chore/code-standard`.
3. Title: `chore: add agreed code standard and CI build`.
4. Description — use this structure for every PR from now on:

```markdown
## What
- Spotless with google-java-format, bound to `validate`; generated code formatted
- `.editorconfig`, LF line endings in `.gitattributes`
- `CODESTYLE.md` as agreed in lesson 02
- GitHub Actions workflow running `./mvnw -B verify` with Java 25 (tests use Testcontainers)

## Why
Lesson 02: the group agreed on a code standard; CI enforces it.

## How to test
With Docker Desktop running: `./mvnw verify` — expect BUILD SUCCESS.
```

5. Click **Create pull request**.

**Check it works:** the PR page shows **"CI / Build and verify"** running (yellow dot), then a green
check after one or two minutes (the first run downloads all dependencies; later runs use the cache).
Click **Details** to see each step's log.

---

## Step 8 — Break it on purpose: red, then green

**Why:** a check that has never been red has never been tested. You need to see the gate close.

In `src/main/java/ee/ta25/billable/common/HomeController.java`, make the `home` method ugly:

```java
// src/main/java/ee/ta25/billable/common/HomeController.java — deliberately badly formatted (temporary)
  @GetMapping("/")
  public String home(Model model){
      model.addAttribute("today",LocalDate.now(TALLINN));
    return "home";}
```

It still compiles and works. Commit and push **without** running Spotless:

```bash
git commit -am "chore: break formatting on purpose to test CI"
git push
```

(If you already installed a pre-commit hook from Task 3, use `git commit --no-verify` this one time —
which also proves why the hook cannot be the gate.)

**Check it works:** the PR check turns **red**. Click **Details** → the "Build, check formatting and run
tests" step:

```
[ERROR] Failed to execute goal com.diffplug.spotless:spotless-maven-plugin:…:check (default)
  on project billable: The following files had format violations:
[ERROR]     src/main/java/ee/ta25/billable/common/HomeController.java
[ERROR]         @@ -12,8 +12,8 @@
…
[ERROR] Run 'mvn spotless:apply' to fix these violations.
Error: Process completed with exit code 1.
```

Note what did **not** happen: the compiler and the tests never ran. The build stopped in `validate`,
the first phase.

Fix it the normal way:

```bash
./mvnw spotless:apply
git diff
git commit -am "style: fix formatting in HomeController"
git push
```

**Check it works:** `git diff` shows Spotless restored the original layout; the new CI run is **green**.
The PR's commit list shows a red ✗ next to the "break formatting" commit and a green ✓ next to the fix.
This red-then-green pair is evidence for ÕV7 — keep it (with "squash and merge", the individual commits
stay visible in the closed pull request).

---

## Step 9 — Review and merge

In class, swap with your neighbour: open their PR (they add you as a collaborator — see Task 1), read
**Files changed** and leave one comment that is *not* about formatting. For example on
`application.properties`: "The password `secret` is written in the file as the default. Should
`CODESTYLE.md` say that real passwords must always come from `DB_PASSWORD`?"

Then merge your own PR:

1. Click **Squash and merge** (or **Create a merge commit**, whichever your group voted for).
2. Check the commit message: `chore: add agreed code standard and CI build (#1)`.
3. **Confirm**, then **Delete branch**.

Update your laptop:

```bash
git switch main
git pull
git branch -d chore/code-standard
```

**Check it works:** `git log --oneline -5` shows the merged work; the Actions tab shows a green run for
the push to `main`.

---

## Step 10 — Protect `main` (if available)

**Why:** so far the red check only *warns*. Branch protection makes GitHub refuse to merge while CI is
red.

On GitHub: **Settings → Branches → Add branch protection rule** (newer interface: **Settings → Rules →
Rulesets → New branch ruleset**).

- Target: `main`
- Require a pull request before merging
- Require status checks to pass → select **Build and verify** (it appears after it has run once)

**Note:** on a *private* repository with a free GitHub account, these rules may not be enforced. They
work on public repositories and with GitHub Pro, which is free for students through the GitHub Student
Developer Pack. If you cannot enable it, "no merge while red" is still in `CODESTYLE.md`, and you follow
it by hand.

---

## Troubleshooting

| Error / symptom | Cause | Fix |
|---|---|---|
| `Could not resolve plugin com.diffplug.spotless:spotless-maven-plugin:REPLACE_WITH_LATEST_VERSION` | The placeholder was not replaced | Put the real latest version from Maven Central into `<spotless.version>`. |
| `NoSuchMethodError: com.sun.tools.javac…` or `IllegalAccessError … does not export com.sun.tools.javac…` | Spotless / google-java-format too old for Java 25 | Update `<spotless.version>` to the latest release. Do not add a `<version>` inside `<googleJavaFormat/>` unless you know it supports Java 25. |
| `spotless:check` fails again right after `spotless:apply` | Line endings: Windows editor writes CRLF | Add `.editorconfig` and the `.gitattributes` line (Step 4); run `git add --renormalize .` once. |
| CI: `./mvnw: Permission denied` | `mvnw` not executable in Git (committed from Windows) | `git update-index --chmod=+x mvnw`, commit, push. |
| CI: `Connection to localhost:5432 refused` in `contextLoads` | The test does not use the container: `@Import(TestcontainersConfiguration.class)` is missing | Compare `BillableApplicationTests` with lesson 01, Step 9. Do not add a `services:` block to hide it. |
| CI: `Could not find a valid Docker environment` | The job does not run on a Linux runner with Docker | Keep `runs-on: ubuntu-latest`. (GitHub's macOS runners have no Docker, and its Windows runners cannot run the Linux `postgres` image.) |
| CI: `release version 25 not supported` | `setup-java` step missing or wrong `java-version` | `java-version: '25'` in quotes, `distribution: temurin`. |
| Local `./mvnw verify` fails in `contextLoads` with `Could not find a valid Docker environment` | Docker Desktop is not running (Testcontainers needs it) | Start Docker Desktop; `docker ps` must work. |
| The workflow never runs | Wrong path or YAML error | File must be `.github/workflows/ci.yml`; YAML uses 2 spaces, no tabs. GitHub shows YAML errors in the Actions tab. |
| IntelliJ keeps re-formatting differently | IntelliJ's formatter ≠ google-java-format | Install the google-java-format plugin, or disable "Reformat code" on save and run `spotless:apply`. |
| Status check not listed in branch protection | It has never run | Push once so the workflow runs; search for `Build and verify`. |

---

## Recap

- `pom.xml` — Spotless plugin with google-java-format, `check` bound to `validate`; version in a
  property, taken from Maven Central.
- `./mvnw spotless:apply` fixes, `./mvnw spotless:check` checks, and every build checks automatically.
- `.editorconfig` and `.gitattributes` — same encoding, indentation and LF line endings everywhere.
- `CODESTYLE.md` — the agreement: date, tools and commands, language, naming, rules, commits, review
  process, and the vote results.
- `.github/workflows/ci.yml` — Temurin 25 and `./mvnw -B verify` on pushes to `main` and on every
  pull request. No database service: `contextLoads` starts its own PostgreSQL with Testcontainers,
  because the `ubuntu-latest` runner has Docker.
- Branch `chore/code-standard` → PR → red (deliberate) → green → reviewed → merged.

---

## Independent work — solutions

> **The tasks are not here.** The task descriptions (Basic / Intermediate / Advanced, with acceptance
> criteria) are in this lesson's [README.md → Independent work (~2 h)](README.md#independent-work-2-h).
> Read them there first and build the features in your project. This section only contains the
> solutions — try each task yourself, then open a solution to compare or when you are stuck.

<details>
<summary>Task 1 — A reviewed pull request (Basic)</summary>

**Give your reviewer access.** On a private repository: **Settings → Collaborators → Add people**. On
the PR page, click the gear next to **Reviewers** and select them.

**Example PR:** branch `feature/request-timing`, implementing lesson 01 Task 4 (the request timing
filter). Title: `feat(common): log duration of every request`.

**Examples of useful, non-formatting review comments:**

1. On `RequestTimingFilter.java`: "Static resources (`/favicon.ico`, CSS) are logged too. Should we skip
   them with `shouldNotFilter()`, so the log shows only page requests?"
2. On the `log.info(...)` line: "At INFO level this writes a line for every request in production.
   Would DEBUG be better, or is this on purpose?"
3. On `millis`: "Integer division drops everything under 1 ms, so fast requests show `0 ms`. Is that
   fine, or should we log microseconds?"
4. On the PR description: "How do I test this? Which log lines should I expect?"

**Replying:** for each comment, push a commit and reply "Done in `3f2a1bc`", or explain your decision:
"I kept INFO on purpose for now — added a TODO for lesson 14." Then **Resolve conversation**.

**Merge** with the method from `CODESTYLE.md`, delete the branch, `git switch main && git pull`.

</details>

<details>
<summary>Task 2 — Clean commit history habits (Basic)</summary>

A practice run on a branch:

```bash
git switch main && git pull
git switch -c docs/readme-how-to-run
```

**Partial staging with `git add -p`.** Make two unrelated changes to `README.md` (a "How to run"
section and a typo fix elsewhere). Then:

```bash
git add -p README.md
```

Git shows each changed block ("hunk") and asks `Stage this hunk [y,n,q,a,d,s,e,?]?`. Press `y` for the
"How to run" hunk and `n` for the typo:

```bash
git commit -m "docs: add how to run section to readme"
git add README.md
git commit -m "docs: fix typo in readme intro"
```

**Fix the last message with `--amend`:**

```bash
git commit --amend -m "docs: fix typo in project description"
```

`--amend` replaces the **last** commit. Do this only before pushing.

**Combine a fix-up commit.** You notice a mistake in the "How to run" section:

```bash
# edit README.md, then:
git add README.md
git commit --fixup=HEAD~1
git rebase -i --autosquash main
```

`--fixup=HEAD~1` creates a commit `fixup! docs: add how to run section to readme`. The interactive
rebase opens an editor; `--autosquash` has already placed the fix-up under its target and marked it
`fixup`. Save and close. (By hand: `git rebase -i main`, change `pick` to `fixup` in front of the commit
to merge into the line above, save, close.)

**Check:** `git log --oneline main..HEAD` shows two clean commits. Push and open the PR.

**Rule:** rewrite history only on **your own branch**, before others build on it. After rewriting a
pushed branch, use `git push --force-with-lease`. Never rewrite `main`.

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

Git looks for hooks in `.git/hooks/` by default, which is **not** part of the repository. We put the
hook in a committed folder and point Git at it.

```sh
#!/bin/sh
# hooks/pre-commit
# Refuses the commit if Spotless finds formatting problems.
# Activate once per clone:  git config core.hooksPath hooks

if ! ./mvnw -q spotless:check > /dev/null 2>&1; then
  echo ""
  echo "✗ Code style check failed."
  echo "  Run:  ./mvnw spotless:apply"
  echo "  Then add the changes and commit again."
  echo ""
  exit 1
fi

exit 0
```

```bash
chmod +x hooks/pre-commit
git config core.hooksPath hooks
git add hooks/pre-commit
git update-index --chmod=+x hooks/pre-commit
git commit -m "chore: add shared pre-commit hook for Spotless"
```

**What happens:**

1. Git runs `hooks/pre-commit` before each commit; a non-zero exit code aborts the commit.
2. `./mvnw -q spotless:check` runs only the Spotless goal (not the whole build), quietly. Maven's start
   takes a few seconds — acceptable for a commit, far faster than waiting for CI.
3. `core.hooksPath` is stored in `.git/config`, which is local, so each teammate runs the `git config`
   command once after cloning. That is why it goes into `CODESTYLE.md`.
4. Git for Windows runs `sh` hooks with its bundled shell; `update-index --chmod=+x` stores the
   executable bit for Linux and macOS users.

**Test it:** break the formatting in `HomeController.java`, run `git commit -am "test"` → refused. Run
`./mvnw spotless:apply`, commit again → works.

**Add to `CODESTYLE.md`** (section 1):

```markdown
<!-- CODESTYLE.md — add to section 1 -->
**Pre-commit hook (optional, recommended):** run `git config core.hooksPath hooks` once after cloning.
The hook only gives faster feedback: it runs on your machine, can be skipped with `--no-verify`, and is
not installed automatically. CI is the gate that nothing can skip.
```

</details>

<details>
<summary>Task 4 — Checkstyle in CI (Advanced)</summary>

Spotless already controls layout, so Checkstyle must **not** check formatting again (two tools with
different opinions on indentation would fight). We give it a small rule set of things a formatter
cannot fix.

**Versions.** Look up the latest versions on Maven Central:
`org.apache.maven.plugins:maven-checkstyle-plugin` and `com.puppycrawl.tools:checkstyle`. The plugin
ships with an old Checkstyle engine by default that does not understand modern Java syntax well, so we
set the engine version explicitly.

Add two properties to `<properties>` in `pom.xml`:

```xml
<!-- pom.xml — inside <properties> -->
<checkstyle-plugin.version>REPLACE_WITH_LATEST_VERSION</checkstyle-plugin.version>
<checkstyle-engine.version>REPLACE_WITH_LATEST_VERSION</checkstyle-engine.version>
```

Add the plugin after Spotless in `<build><plugins>`:

```xml
<!-- pom.xml — inside <build><plugins>, after the Spotless plugin -->
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-checkstyle-plugin</artifactId>
	<version>${checkstyle-plugin.version}</version>
	<dependencies>
		<dependency>
			<groupId>com.puppycrawl.tools</groupId>
			<artifactId>checkstyle</artifactId>
			<version>${checkstyle-engine.version}</version>
		</dependency>
	</dependencies>
	<configuration>
		<configLocation>checkstyle.xml</configLocation>
		<consoleOutput>true</consoleOutput>
		<failOnViolation>true</failOnViolation>
		<violationSeverity>warning</violationSeverity>
		<includeTestSourceDirectory>true</includeTestSourceDirectory>
	</configuration>
	<executions>
		<execution>
			<id>checkstyle</id>
			<phase>verify</phase>
			<goals>
				<goal>check</goal>
			</goals>
		</execution>
	</executions>
</plugin>
```

Create `checkstyle.xml` in the project root:

```xml
<?xml version="1.0"?>
<!-- checkstyle.xml — rules a formatter cannot enforce. Layout is Spotless's job. -->
<!DOCTYPE module PUBLIC
    "-//Checkstyle//DTD Checkstyle Configuration 1.3//EN"
    "https://checkstyle.org/dtds/configuration_1_3.dtd">
<module name="Checker">
  <property name="severity" value="error"/>

  <module name="TreeWalker">
    <!-- Imports -->
    <module name="AvoidStarImport"/>
    <module name="UnusedImports"/>
    <module name="IllegalImport">
      <property name="illegalPkgs" value="javax.persistence, javax.validation, javax.servlet"/>
    </module>

    <!-- Suspicious code -->
    <module name="EmptyCatchBlock"/>
    <module name="EqualsHashCode"/>
    <module name="StringLiteralEquality"/>
    <module name="SimplifyBooleanExpression"/>
    <module name="SimplifyBooleanReturn"/>

    <!-- Size limits that force smaller methods -->
    <module name="MethodLength">
      <property name="max" value="40"/>
    </module>
    <module name="ParameterNumber">
      <property name="max" value="5"/>
    </module>

    <!-- Use a logger -->
    <module name="Regexp">
      <property name="format" value="System\.(out|err)\.print"/>
      <property name="illegalPattern" value="true"/>
      <property name="message" value="Use an SLF4J logger instead of System.out/System.err."/>
    </module>
  </module>
</module>
```

**What happens:**

1. The plugin runs Checkstyle's `check` goal in the `verify` phase — after the tests, so `./mvnw verify`
   (and therefore CI) runs it with no workflow change.
2. `failOnViolation` + `violationSeverity=warning` — any violation fails the build.
3. `IllegalImport` catches `javax.persistence` and friends: code copied from pre-Boot-3 tutorials. A
   formatter would happily format that code; Checkstyle refuses it.
4. `StringLiteralEquality` catches `name == "Acme"` (compares references, not text) — a real bug.
5. `MethodLength` and `ParameterNumber` are design limits: when a controller method passes 40 lines,
   it is a sign that logic belongs in a service (lesson 07).
6. `Regexp` forbids `System.out.print…` — the rule from `CODESTYLE.md` section 4, now checked by a
   machine.

**Check it works:**

```bash
./mvnw checkstyle:check
```

prints `You have 0 Checkstyle violations.` Then add `System.out.println("hi");` to `HomeController.home()`,
run `./mvnw verify` → `[ERROR] … HomeController.java:[…] Use an SLF4J logger instead of System.out/System.err.`
and `BUILD FAILURE`. Remove it again.

**Alternative: Error Prone.** Error Prone is a compiler plugin from Google that finds bug patterns during
compilation (for example ignored return values). It needs extra compiler flags and JVM options on
modern JDKs; follow its official installation page for Maven if you prefer it over Checkstyle.

**Add to `CODESTYLE.md`** (section 1, tools table):

```markdown
| Static checks (CI, in `verify`) | Checkstyle (`checkstyle.xml`: imports, suspicious code, size limits, no System.out) | `./mvnw checkstyle:check` |
```

```bash
git add pom.xml checkstyle.xml CODESTYLE.md
git commit -m "ci: add checkstyle rules for imports, bug patterns and method size"
```

</details>
