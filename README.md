# M8 — Programming: Frameworks and Architectural Patterns

**M8. Programmeerimine: Raamistikud ja arhitektuurimustrid**

Student guide for the module. You will build one real web application — **Billable**, a time-tracking
and invoicing tool — in a web framework of your choice, and along the way learn the patterns,
architecture, testing and documentation habits that separate "it works on my machine" from software a
team can maintain.

| | |
|---|---|
| Length | 15 lessons × 2 h in class |
| Independent work | ~2 h after each lesson, ~35 h in total ([plan](#planning-your-independent-work-35-h)) |
| Tracks | **Laravel** (PHP) or **Spring Boot** (Java) — you pick one |
| Before you start | [PHP OOP fundamentals](https://github.com/kuressaareametikool/VILT-crud-cycle/blob/main/php-oop-lesson-guide.md) |
| Assessment | [Capstone brief and rubric](CAPSTONE.md) |

---

## How this module works

You study next to a job, so the module is built around a few rules that save your time.

1. **One project, start to finish.** Every lesson adds a piece to the same application,
   [Billable](PROJECT.md). Nothing you build is thrown away. The finished project is your capstone.
2. **The teacher codes, you code along.** In class the teacher builds the lesson's feature live,
   following the same guide you are reading. Type along in your own project. If you fall behind, the
   guide has every step — you do not need to copy from the screen.
3. **Every lesson is 2 hours and one topic.** Each lesson folder has a concept page and a step-by-step
   walkthrough for your framework.
4. **Independent work finishes the lesson.** Each lesson ends with ~2 hours of tasks you do at home. They
   add the features the class did not build, so your project is complete before the next session.
5. **Commit as you go.** Your Git history is evidence for several learning outcomes. Small commits with
   clear messages, every lesson.

### Missed a session?

Open that lesson's walkthrough and follow it from the top. Each walkthrough starts from the state the
previous lesson ended in (see [what exists after each lesson](PROJECT.md#what-exists-after-each-lesson)),
so you can catch up in order. If you are more than one lesson behind, tell the teacher — catching up is
normal, silently falling behind is not.

---

## Choose your track

Both tracks build exactly the same application with the same database, the same features and the same
tests. The concepts are identical; only the syntax differs.

| | Laravel | Spring Boot |
|---|---|---|
| Language | PHP 8.4 | Java 25 |
| Framework | Laravel 12 or newer | Spring Boot 4.x (Maven) |
| Views | Blade | Thymeleaf |
| ORM | Eloquent (Active Record) | JPA / Hibernate (Data Mapper) |
| Migrations | Laravel migrations | Flyway |
| Tests | PHPUnit, Mockery | JUnit 5, Mockito, AssertJ |
| Code standard | Laravel Pint (PSR-12 based) | Spotless + Google Java Format |
| Database | PostgreSQL 17 in Docker | PostgreSQL 17 in Docker |

**Pick Laravel** if you did the PHP OOP guide and want the fastest route to a working app.
**Pick Spring Boot** if you want to work in Java or your workplace uses it. It is more verbose and more
explicit — you see more of the machinery.

Pick one and stay with it. Reading the other track's walkthrough now and then is a good way to see which
ideas are universal and which are just one framework's habit.

---

## Lessons

| # | Lesson | Outcomes | Laravel | Spring Boot |
|---|---|---|---|---|
| 01 | [Frameworks and the request lifecycle](lessons/01-frameworks-and-request-lifecycle/) | ÕV3 | [→](lessons/01-frameworks-and-request-lifecycle/laravel.md) | [→](lessons/01-frameworks-and-request-lifecycle/spring-boot.md) |
| 02 | [An agreed code standard](lessons/02-code-standard/) | ÕV7 | [→](lessons/02-code-standard/laravel.md) | [→](lessons/02-code-standard/spring-boot.md) |
| 03 | [ORM basics: models and migrations](lessons/03-orm-basics/) | ÕV4 | [→](lessons/03-orm-basics/laravel.md) | [→](lessons/03-orm-basics/spring-boot.md) |
| 04 | [MVC: controllers and views](lessons/04-mvc-reading-data/) | ÕV3 | [→](lessons/04-mvc-reading-data/laravel.md) | [→](lessons/04-mvc-reading-data/spring-boot.md) |
| 05 | [MVC: forms and validation](lessons/05-mvc-forms-and-validation/) | ÕV3 | [→](lessons/05-mvc-forms-and-validation/laravel.md) | [→](lessons/05-mvc-forms-and-validation/spring-boot.md) |
| 06 | [ORM: relationships and the N+1 problem](lessons/06-orm-relationships/) | ÕV4 | [→](lessons/06-orm-relationships/laravel.md) | [→](lessons/06-orm-relationships/spring-boot.md) |
| 07 | [Layers and dependency injection](lessons/07-layers-and-dependency-injection/) | ÕV1, ÕV3 | [→](lessons/07-layers-and-dependency-injection/laravel.md) | [→](lessons/07-layers-and-dependency-injection/spring-boot.md) |
| 08 | [Patterns I: Strategy and Repository](lessons/08-patterns-strategy-and-repository/) | ÕV1 | [→](lessons/08-patterns-strategy-and-repository/laravel.md) | [→](lessons/08-patterns-strategy-and-repository/spring-boot.md) |
| 09 | [Patterns II: Observer, Adapter, Decorator](lessons/09-patterns-events-and-adapters/) | ÕV1 | [→](lessons/09-patterns-events-and-adapters/laravel.md) | [→](lessons/09-patterns-events-and-adapters/spring-boot.md) |
| 10 | [Math and logic in applications](lessons/10-math-and-logic/) | ÕV2 | [→](lessons/10-math-and-logic/laravel.md) | [→](lessons/10-math-and-logic/spring-boot.md) |
| 11 | [Unit testing](lessons/11-unit-testing/) | ÕV5 | [→](lessons/11-unit-testing/laravel.md) | [→](lessons/11-unit-testing/spring-boot.md) |
| 12 | [Mocks and test doubles](lessons/12-mocks-and-test-doubles/) | ÕV6 | [→](lessons/12-mocks-and-test-doubles/laravel.md) | [→](lessons/12-mocks-and-test-doubles/spring-boot.md) |
| 13 | [A complex component: invoice generation](lessons/13-complex-component-invoicing/) | ÕV8, ÕV2 | [→](lessons/13-complex-component-invoicing/laravel.md) | [→](lessons/13-complex-component-invoicing/spring-boot.md) |
| 14 | [Integration tests and data integrity](lessons/14-integration-tests-and-data-integrity/) | ÕV4, ÕV5, ÕV8 | [→](lessons/14-integration-tests-and-data-integrity/laravel.md) | [→](lessons/14-integration-tests-and-data-integrity/spring-boot.md) |
| 15 | [Documentation in English](lessons/15-documentation-in-english/) | ÕV9 | [→](lessons/15-documentation-in-english/laravel.md) | [→](lessons/15-documentation-in-english/spring-boot.md) |

The order is **build order**, not outcome order: an application has to exist before it can be tested,
refactored into patterns or documented.

### How each lesson is organised

Each lesson folder contains three files:

| File | What it is | When to read it |
|---|---|---|
| `README.md` | **Concepts.** Why the topic matters, the ideas and rules, common mistakes, independent work, self-check, checklist | Before class (10 min skim) and again when doing the independent work |
| `laravel.md` | **Walkthrough.** Every step we build in class, with complete code and explanations, plus solutions | During class and when catching up |
| `spring-boot.md` | Same walkthrough for Spring Boot | During class and when catching up |

Independent-work solutions are at the bottom of each walkthrough, collapsed. Try first; open them when
you are stuck or to compare after you finish.

---

## Learning outcomes

| Code | Outcome (ET) | In English | Lessons |
|---|---|---|---|
| ÕV1 | tunneb enamlevinud programmeerimismustreid | knows the most common programming patterns | 07, 08, 09 |
| ÕV2 | kasutab rakenduste koostamisel matemaatika- ja loogikafunktsioone | uses mathematical and logical functions when building applications | 10, 13 |
| ÕV3 | realiseerib rakenduse MVC arhitektuuriga rakendusena | implements an application using MVC architecture | 01, 04, 05, 07 |
| ÕV4 | kasutab parimate praktikate kohaselt ORM vahendeid | uses ORM tools according to best practices | 03, 06, 14 |
| ÕV5 | mõistab ühiktestide olemust ning nende kasutamisvõimalusi | understands the nature and uses of unit tests | 11, 14 |
| ÕV6 | kasutab testides mock-klasse | uses mock classes in tests | 12 |
| ÕV7 | kasutab korrektselt kokkulepitud koodistandardit | correctly applies an agreed code standard | 02, then every lesson |
| ÕV8 | loob suurema keerukusastmega rakendusi… | builds more complex applications, including mathematically and logically complex algorithms and components | 13, 14 |
| ÕV9 | dokumenteerib loodud rakendused inglise keeles | documents the applications in English | 15, then every commit |

---

## Planning your independent work (35 h)

| Block | Hours | What |
|---|---|---|
| Independent work after lessons 01–14 | 28 | ~2 h per lesson — finish the lesson's features in your project |
| Documentation (after lesson 15) | 3 | README, ADRs, diagrams, final English pass |
| Capstone extension feature | 2 | One feature of your own choice from the [capstone brief](CAPSTONE.md) |
| Defence preparation | 2 | Rehearse the demo, re-read your own code |

**Tip for working students:** two short sessions a week (e.g. 1 h on a weekday evening + 1 h at the
weekend) work better than one long one before class. The independent tasks are split into **basic**,
**intermediate** and **advanced** — basic tasks are required to keep the project on track; the others
raise your grade.

---

## What you need

Install before lesson 01. Lesson 01's walkthrough has details and troubleshooting.

| Everyone | Laravel track | Spring Boot track |
|---|---|---|
| Git + a GitHub account | PHP 8.4 and Composer ([Laravel Herd](https://herd.laravel.com) is the easiest route on Windows/macOS) | JDK 25 (e.g. Eclipse Temurin) |
| Docker Desktop (runs PostgreSQL) | Laravel installer (`composer global require laravel/installer`) | IntelliJ IDEA (Community is enough) or VS Code with Java extensions |
| A code editor (VS Code, PhpStorm, IntelliJ) | | Maven comes with the project (`./mvnw`) |

---

## Using AI assistants

You may use AI assistants — you will at work. The rules:

- **You must be able to explain every line you submit.** The defence will check this.
- Frameworks change fast. AI tools (and most tutorials) often produce code for **older versions**.
  Check that a suggestion matches the version in this guide before pasting it.
- Keep a short `docs/ai-usage.md`: what you used AI for and one thing it got wrong.

---

## Further reading

- [PROJECT.md](PROJECT.md) — the Billable domain, business rules, data model and code map
- [CAPSTONE.md](CAPSTONE.md) — what you submit and how it is assessed
- [PHP OOP fundamentals](https://github.com/kuressaareametikool/VILT-crud-cycle/blob/main/php-oop-lesson-guide.md) — start here if classes and interfaces feel shaky
- [Laravel request lifecycle (VILT)](https://github.com/kuressaareametikool/VILT-crud-cycle) — a deeper look at how data flows through Laravel
