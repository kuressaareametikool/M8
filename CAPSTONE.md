# Capstone: Brief and Assessment

**All outcomes (ÕV1–ÕV9).** Every lesson is practice for this.

Your capstone is the **Billable** application you build throughout the module, plus **one extension
feature** of your own choice. You do not start a new project at the end — if you kept up with the
independent work, most of the capstone already exists by lesson 13.

---

## What you submit

A link to your Git repository (GitHub or similar, shared with the teacher) containing:

| Deliverable | Covers | Lesson |
|---|---|---|
| Full Git history with consistent commit messages | ÕV7 | 02 → |
| `CODESTYLE.md` and a formatter enforced in CI | ÕV7 | 02 |
| Working MVC application: clients, projects, time entries, invoices | ÕV3 | 01, 04, 05, 07 |
| Migrations for the whole schema; relationships; no N+1 on list pages | ÕV4 | 03, 06, 14 |
| Billing strategies, notifier adapter, budget-warning observer; `docs/patterns.md` | ÕV1 | 07–09 |
| `Money`, `TimeRounding`, `DueDateCalculator`; VAT and rounding correct | ÕV2 | 10 |
| Invoice generation with overlap detection and gap-free numbering; `docs/algorithm.md` | ÕV8 | 13, 14 |
| Unit tests (15+), mock-based tests, at least three feature/integration tests; all green in CI | ÕV5, ÕV6 | 11, 12, 14 |
| `README.md`, doc comments on the domain code, ER + sequence diagrams, three ADRs — in English | ÕV9 | 15 |
| One extension feature (see below) with tests and an ADR | ÕV8 | — |
| `docs/ai-usage.md` | — | — |

### Hard constraints

- No `float`/`double` anywhere money is calculated.
- No business logic in controllers or views.
- Every schema change is a migration. Never edit the database by hand.
- The test suite passes on a fresh clone, following only your README.
- All code, comments, commit messages and documentation in English.

---

## Extension feature — pick one

Choose one, or propose your own of similar size to the teacher before lesson 13.

| Feature | What makes it non-trivial |
|---|---|
| **Credit note** | Reverses (part of) a sent invoice; own numbering series; negative amounts; cannot exceed the original |
| **Multi-currency clients** | Invoice in EUR/USD/GBP with a rate from an external API behind an adapter; rate stored on the invoice; API mocked in tests |
| **Recurring retainer** | Monthly fixed amount plus hourly overage above an included number of hours |
| **Late payment interest** | Daily interest on overdue invoices; correct day counting; decision table for edge cases |
| **CSV import of time entries** | Parse, validate row by row, report errors per line, import all-or-nothing in one transaction |
| **Utilisation report** | Billable vs non-billable hours per week and per tag, with percentages that add up to 100 % |
| **PDF invoice** | Generated from a template; totals identical to the screen; file name uses the invoice number |

---

## The defence

15 minutes per student: **7 minutes demo, 8 minutes questions**.

**Demo:** generate an invoice for a client from real entries, show one validation error, show your
extension feature, run the test suite.

Expect questions like:

- "Open the file where the billing strategy is chosen. Why is it there and not in the controller?"
- "What SQL does this line run? How many queries does the time sheet page make?"
- "Why are amounts integers? Show me where rounding happens."
- "This test uses a mock. What would break if you used the real class?"
- "What happens if two people generate an invoice at the same second?"
- "Explain this line." (Any line. That is why you must understand your AI-assisted code.)

The defence is where partial outcomes are confirmed — or not.

---

## Assessment rubric

Each outcome is assessed separately as **not yet / achieved / exceeds**.
**To pass, all nine outcomes must be at least *achieved*.** *Exceeds* raises the grade but never makes up
for an outcome that is *not yet*.

| Outcome | Not yet | Achieved | Exceeds |
|---|---|---|---|
| **ÕV1** Patterns | Names patterns but cannot find them in own code | Strategy, Observer, Adapter and DI used correctly; each explained in own words in `docs/patterns.md` | Uses a pattern not taught (with reasons), and shows an anti-pattern they refactored away |
| **ÕV2** Math & logic | Floats for money; rounding happens by accident | Integer cents, explicit half-up rounding, VAT and time rounding correct at boundaries; due-date rule correct | Largest-remainder allocation or own statistical function; decision table with every row tested |
| **ÕV3** MVC | Queries or calculations in controllers or views | Thin controllers, logic in services, validation at the edge, correct redirects and status codes | Consistent error pages, route-level authorisation or policies, a clear explanation of the request lifecycle |
| **ÕV4** ORM | Schema edited by hand; N+1 on list pages | Correct relationships, migrations for everything, eager loading where needed, N+1 found and fixed with evidence | Query counts measured before/after, aggregates in SQL rather than in loops, transactions used deliberately |
| **ÕV5** Unit tests | Few tests, or tests that need a database for pure logic | 15+ meaningful unit tests, Arrange–Act–Assert, boundary cases, green in CI | Data-driven boundary tables, a coverage report, and a written judgement of what is left untested and why |
| **ÕV6** Mocks | Mocks copied from a tutorial, cannot explain them | Clock and notifier replaced in tests; asserts both that something happened **and** that it did not | Explains mock vs stub vs fake with own examples; uses a fake where it is better than a mock |
| **ÕV7** Code standard | Inconsistent style; no agreed standard | `CODESTYLE.md` agreed, formatter enforced in CI, consistent commit messages across the history | Static analysis in CI, a reviewed pull request whose comments changed a design decision |
| **ÕV8** Complexity | CRUD only | Invoice generation complete: overlap detection, rounding, VAT, gap-free numbering, one transaction, tested | Extension feature well beyond CRUD, complexity (Big-O) explained, race condition handled and demonstrated |
| **ÕV9** Documentation | Thin README, Estonian mixed into code | README a stranger can follow, doc comments on domain code, ER + sequence diagrams, three ADRs, correct English | A classmate followed your README on a clean machine and you fixed what they got stuck on (notes attached) |

### If a single grade is required

| Component | Weight |
|---|---|
| Working application (ÕV1–ÕV4, ÕV8) | 55 % |
| Tests and code standard (ÕV5–ÕV7) | 25 % |
| Documentation and defence (ÕV9) | 20 % |

The pass condition is still: **all nine outcomes at *achieved***.

---

## If you are short of time

Cut **breadth, not depth**. A project with fewer screens but clean layers, real tests and a correct
invoice beats one with every screen and no tests.

Do not cut lesson 13 (invoice generation) or the testing lessons — they are the hardest outcomes to add
at the last minute, and the defence will go straight to them.
