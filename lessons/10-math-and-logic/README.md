# 10 — Math and logic in applications

**Outcomes:** ÕV2 — *kasutab rakenduste koostamisel matemaatika- ja loogikafunktsioone* · **Time:** 2 h in class + ~2 h independent
**Before this lesson:** your project matches the end state of lesson 09 (see [what exists after each lesson](../../PROJECT.md#what-exists-after-each-lesson))
**Walkthroughs:** [Laravel](laravel.md) · [Spring Boot](spring-boot.md)

---

## Why this matters

"The number on the invoice is wrong" is the bug that customers notice, accountants report and managers
remember. It is rarely a big mistake. It is usually one cent, caused by a `float`, by rounding in the
wrong place, or by a time entry that crossed the night when the clocks changed.

Billable's business rules are full of small calculations:

- money in cents (R1), VAT at 24 % (R2), rounding half up exactly once (R3);
- time rounded **up** to 1, 6 or 15 minutes (R5), hourly amount = minutes × rate ÷ 60 (R6);
- a due date 14 days later that must not fall on a weekend (R12);
- "can this invoice still be edited?" — a question with a yes/no answer that depends on its state (R13).

None of this needs advanced mathematics. It needs *precise* thinking: which number type, which
rounding rule, at which step, and which cases exist. That is what ÕV2 asks for, and the defence
question "why are amounts integers — show me where rounding happens" tests exactly this.

The good news: everything in this lesson is **pure functions** — no database, no HTTP. They are the
easiest code in the project to test, and lesson 11 tests them.

---

## Today's goal

At the end of class your application has:

- a `Money` value object (integer cents) with parsing, arithmetic, percentages in basis points and
  Estonian formatting;
- `TimeRounding::roundUp()` implementing R5 with ceiling division;
- billing strategies that take and return `Money` instead of plain integers;
- **one** place that calculates a project's billable minutes, rounded per entry, used by both the
  project page and `BudgetMonitor`;
- a project page that shows *billable so far*, *VAT 24 %* and *total*, all calculated with `Money`.

By the end you can:

- explain with an example why `0.1 + 0.2 != 0.3`, and why that rules out floats for money;
- name six rounding modes and round a negative number with each;
- show why rounding each line and rounding the total give different results;
- calculate VAT with basis points and integer arithmetic;
- use integer division, modulo and ceiling division correctly, including for negative numbers;
- explain what goes wrong with durations on the days the clocks change in Tallinn;
- turn a rule into a decision table, and a decision table into code and tests.

---

## Concepts

### 1. Why floats fail for money

Run this in both languages:

```php
// PHP (php -a or Tinker)
var_dump(0.1 + 0.2);          // float(0.30000000000000004)
var_dump(0.1 + 0.2 == 0.3);   // bool(false)
echo 0.1 + 0.2;               // 0.3   ← echo rounds for display and hides the problem!
var_dump((int) (19.99 * 100)); // int(1998)   ← one cent lost
```

```java
// Java (jshell)
System.out.println(0.1 + 0.2);              // 0.30000000000000004
System.out.println(0.1 + 0.2 == 0.3);       // false
System.out.println((long) (19.99 * 100));   // 1998   ← one cent lost
```

**Why?** A `float` (PHP) or `double` (Java) stores numbers in **binary** (base 2). In base 2, only
fractions whose denominator is a power of two can be written exactly: ½ = 0.1₂, ¼ = 0.01₂, ⅛ = 0.001₂.
One tenth is not one of them:

```
0.1 (decimal) = 0.000110011001100110011001100110011…  (binary, repeats forever)
```

It is the same problem as writing ⅓ in decimal: 0.333333… never ends, so you must stop somewhere and
the stored value is slightly wrong. A `double` stops after 52 binary digits. So `0.1` is really
0.1000000000000000055511…, `0.2` is really 0.2000000000000000111022…, and their sum is not the number
the computer stores for `0.3`.

`19.99 * 100` becomes 1998.9999999999998, and casting to an integer **cuts off** the decimals — so
1998 cents. Add ten times `0.1` and you get 0.9999999999999999, not 1.

For science and graphics these tiny errors do not matter. For money they do: an invoice must add up to
the cent, every time, on every machine. **R1: all money is integer cents.**

### 2. Three safe ways to represent money

| Representation | Example for 85,50 € | Pros | Cons |
|---|---|---|---|
| **Integer cents** (our choice) | `8550` (`int` in PHP, `long` in Java) | Exact, fast, maps to an SQL `bigint`, same in both tracks | You must remember the unit; division needs explicit rounding |
| **Decimal type** | `new BigDecimal("85.50")` (Java), `new BcMath\Number('85.50')` (PHP 8.4) | Exact, any precision, rounding mode built in | Slower, more verbose; `BigDecimal` has traps (below) |
| **Money library** | `Money::EUR(8550)` (moneyphp), JSR 354 (Java) | Currencies, allocation, formatting | Extra dependency; hides the maths we want to learn |

We use integer cents, wrapped in a `Money` value object (section 10), so nobody can confuse cents with
euros or minutes.

**Limits.** A 64-bit integer holds up to 9 223 372 036 854 775 807 cents — about 92 thousand billion
euros. Enough. But what happens at the limit is different per language:

- **PHP** silently turns an overflowing `int` into a `float`: `PHP_INT_MAX + 1` is
  `9.223372036854776E+18`. Our `Money` checks for this and throws.
- **Java** silently *wraps around*: `Long.MAX_VALUE + 1` is `-9223372036854775808`. Use
  `Math.addExact`, `Math.multiplyExact`, which throw `ArithmeticException` instead.

**`BigDecimal` traps (Java)**, in case you use it at work:

```java
new BigDecimal(0.1)      // 0.1000000000000000055511151231257827… (the double's error is kept!)
new BigDecimal("0.1")    // exactly 0.1 — always construct from a String (or BigDecimal.valueOf)
new BigDecimal("2.0").equals(new BigDecimal("2.00"))      // false — equals compares the scale too
new BigDecimal("2.0").compareTo(new BigDecimal("2.00"))   // 0 — use compareTo for value
new BigDecimal("10").divide(new BigDecimal("3"))          // ArithmeticException: non-terminating
new BigDecimal("10").divide(new BigDecimal("3"), 2, RoundingMode.HALF_UP)   // 3.33
```

**PHP**: the `bcmath` extension works on decimal strings (`bcadd('0.1', '0.2', 2)` → `"0.30"`). The
arithmetic functions truncate to the scale you give them; since PHP 8.4 you round explicitly with
`bcround('16.5', 0, RoundingMode::HalfAwayFromZero)` → `"17"` (also `bcfloor()`, `bcceil()`). PHP 8.4 also
added the `BcMath\Number` object, which supports operators: `new BcMath\Number('85.50') * '0.24'` →
`20.5200`, and `->round(2)`. It is the closest PHP gets to Java's `BigDecimal`. And never do money maths
with `round()`, `floor()` or `number_format()` on floats — they only hide the error for display.

### 3. Rounding modes

When a result has more decimals than you can keep, you must **choose** how to round. Here are the six
modes you will meet, applied to rounding to a whole number:

| Mode | Java `RoundingMode` | 2.4 | 2.5 | 2.6 | 3.5 | −2.5 | −2.6 | Typical use |
|---|---|---|---|---|---|---|---|---|
| **Half up** (half away from zero) | `HALF_UP` | 2 | 3 | 3 | 4 | −3 | −3 | Commercial rounding, invoices, R3 |
| **Half even** (banker's rounding) | `HALF_EVEN` | 2 | 2 | 3 | 4 | −2 | −3 | Statistics, some banking; no upward bias |
| **Up** (away from zero) | `UP` | 3 | 3 | 3 | 4 | −3 | −3 | "Every started hour is billed" |
| **Down** (toward zero, truncation) | `DOWN` | 2 | 2 | 2 | 3 | −2 | −2 | What `(int)` casts and integer division do |
| **Ceiling** (toward +∞) | `CEILING` | 3 | 3 | 3 | 4 | −2 | −2 | Time rounding up (R5) for positive values |
| **Floor** (toward −∞) | `FLOOR` | 2 | 2 | 2 | 3 | −3 | −3 | Age in whole years, "complete weeks" |

Read the table column by column; the negative numbers are where the modes really differ.

- **Half up vs half even.** They differ only when the value is *exactly* halfway. Half up always goes
  away from zero, so across millions of transactions the rounding errors add up in one direction. Half
  even goes to the even neighbour — 2.5 → 2, 3.5 → 4 — so errors cancel out on average. Billable uses
  half up because R3 says so; the important thing is that the choice is written down.
- **What the languages do by default:** PHP's `round()` is half away from zero (`round(-2.5)` =
  `-3`). Other modes are an option: since PHP 8.4 with the `RoundingMode` enum
  (`round(2.5, 0, RoundingMode::HalfEven)` = `2`), before that with constants like
  `PHP_ROUND_HALF_EVEN`. Java's `Math.round()` is *half toward +∞*: `Math.round(-2.5)` is **−2**, not
  −3. That matches none of the rows above for negatives — one more reason to do rounding explicitly.
- **Truncation is rounding too.** `intdiv(7, 2)` and `7 / 2` in Java give 3: they round toward zero.
  Every integer division is a rounding decision, whether you notice it or not.

### 4. Round once, where the fraction appears (R3)

R3 says: *amounts are rounded half up to the cent, once, at the point where a fraction appears.*
"Once" matters, because rounding at different steps gives different answers.

**Worked example.** A project bills 85,00 € per hour. Three entries of 10 minutes each (rounding
increment 1 minute).

One entry is worth 10 × 8500 ÷ 60 = 1 416.666… cents.

| | Round **each line**, then add | Add the minutes, round **once** |
|---|---|---|
| Entry 1 | 1 416.67 → 1 417 | 10 min |
| Entry 2 | 1 416.67 → 1 417 | 10 min |
| Entry 3 | 1 416.67 → 1 417 | 10 min |
| Result | **4 251 cents = 42,51 €** | 30 × 8500 ÷ 60 = 4 250 → **42,50 €** |

One cent apart. Neither is "wrong" mathematically — but only one can be on the invoice, and it must be
the same every time. Billable's decisions, which follow from R3, R5 and R6:

1. **Time** is rounded **per entry** (R5 says "each entry"): 7 minutes with a 15-minute increment is
   billed as 15. Minutes are whole numbers, so this creates no money fractions.
2. **Money** fractions appear in exactly two places. The first is the hourly amount: minutes × hourly
   rate ÷ 60. We add up the rounded minutes of all entries of a project first, then do this division
   **once** per project (one invoice line in lesson 13), rounding half up.
3. The second is **VAT**, calculated **once**, on the subtotal of the whole invoice — not per line, and
   rounded half up with the same function as the hourly amount.
4. **Totals** are sums of already-rounded amounts, so they need no further rounding. That guarantees
   that the lines on the invoice add up to the subtotal the customer sees.

The general lesson: decide where rounding happens, write it down (a comment, an ADR), and do it with
exactly one rounding function in the code (our `Money` has `divideHalfUp()`).

### 5. Percentages and basis points

A percentage is a fraction of 100. To keep it an integer, we store rates in **basis points** (bp):
one hundredth of a percent.

| Rate | Basis points | Stored as |
|---|---|---|
| 1 % | 100 bp | `100` |
| 24 % (Estonian VAT since 1 July 2025) | 2400 bp | `2400` |
| 22 % (the rate before that) | 2200 bp | `2200` |
| 8,5 % | 850 bp | `850` |
| 80 % (budget warning threshold) | 8000 bp | `8000` |

A percentage of an amount is then:

```
part = amount × bp ÷ 10 000
```

and with half-up rounding in integer arithmetic (for non-negative amounts):

```
part = (amount × bp + 5 000) div 10 000
```

Adding half of the divisor before dividing turns "round toward zero" into "round half up". The same
trick works for any divisor `d`: `(n + d div 2) div d`. For negative `n` (credit notes later) you round
the absolute value and put the sign back; our `Money` does this.

The invoice table stores `vat_rate_bp` (R2), because rates change: an invoice from June 2025 had 22 %,
and it must still show 22 % when you open it in 2027.

### 6. VAT worked example

Subtotal: **1 234,56 €** = 123 456 cents. VAT rate: 24 % = 2400 bp.

```
VAT   = (123 456 × 2 400 + 5 000) div 10 000
      = (296 294 400 + 5 000) div 10 000
      = 296 299 400 div 10 000
      = 29 629 cents                     (the exact value was 29 629.44)
      = 296,29 €
Total = 123 456 + 29 629 = 153 085 cents = 1 530,85 €
```

A curious fact: with 24 % and whole cents, the exact result can **never** end in exactly .5 of a cent
(`amount × 24` is always even, so `amount × 24 ÷ 100` never has ",5" at the end). The half-up rule
still matters for other rates: 0,75 € × 22 % = 16,5 cents → **17** with half up, **16** with half even.
Code must work for every rate, not only today's.

### 7. Integer division and modulo

Integer division returns only the whole part; **modulo** (`%`) returns the remainder.

| Expression | PHP | Java | Result |
|---|---|---|---|
| 750 minutes → hours | `intdiv(750, 60)` | `750 / 60` | 12 |
| …and remaining minutes | `750 % 60` | `750 % 60` | 30 → "12 h 30 min" |
| −7 ÷ 2 (toward zero) | `intdiv(-7, 2)` | `-7 / 2` | −3 |
| −7 remainder | `-7 % 2` | `-7 % 2` | −1 |
| −7 ÷ 2 (toward −∞) | `(int) floor(-7 / 2)` | `Math.floorDiv(-7, 2)` | −4 |
| −7 modulo (always ≥ 0) | — | `Math.floorMod(-7, 2)` | 1 |

Watch out:

- In PHP, `7 / 2` is `3.5` (a float!). Use `intdiv()` when you want an integer.
- In Java, `7 / 2` is `3` because both operands are `int`. `7 / 2.0` is `3.5`.
- Both languages round integer division **toward zero**, and `%` takes the sign of the left operand.
  `-7 % 2` is `-1`, so `$n % 2 === 1` is **not** a correct "is odd" test for negative numbers. Java's
  `Math.floorMod` always returns a non-negative result for a positive divisor — use it for "day of
  week" style calculations.

**Ceiling division** — rounding a division *up* — is needed for R5:

```
ceil(a ÷ b) = (a + b − 1) div b        for a ≥ 0, b > 0
```

Why it works: adding `b − 1` pushes every value that is not an exact multiple of `b` past the next
multiple, but an exact multiple stays just below the next one.

Rounding minutes up to an increment is ceiling division followed by multiplying back:

```
roundUp(minutes, inc) = (minutes + inc − 1) div inc × inc
```

| Minutes | Increment 15 | Calculation | Result |
|---|---|---|---|
| 0 | 15 | (0 + 14) div 15 × 15 | 0 |
| 1 | 15 | (1 + 14) div 15 × 15 | 15 |
| 14 | 15 | (14 + 14) div 15 × 15 | 15 |
| 15 | 15 | (15 + 14) div 15 × 15 | 15 |
| 16 | 15 | (16 + 14) div 15 × 15 | 30 |
| 7 | 6 | (7 + 5) div 6 × 6 | 12 |

These rows are the boundary cases you will test in lesson 11: zero, just above a boundary, just below,
exactly on it.

Why not `ceil($minutes / $increment)`? In PHP that mixes in a float (`ceil()` returns `float`), and in
Java `Math.ceil(7 / 15)` is `0` because `7 / 15` is already integer division. The integer formula has
no such traps. Java 18+ has it built in: `Math.ceilDiv(a, b)` is integer division rounded up, so the
Spring walkthrough writes `Math.ceilDiv(minutes, inc) * inc`. PHP has no such function, so the Laravel
walkthrough uses the formula.

### 8. Durations, dates and time zones

**Minutes between two timestamps.** Subtract two *instants* (points on the global timeline), never two
wall-clock readings:

| Language | Correct | Watch out |
|---|---|---|
| PHP | `intdiv($end->getTimestamp() - $start->getTimestamp(), 60)` | Carbon 3's `diffInMinutes()` returns a **float** and is **signed** (negative if the order is reversed) |
| Java | `Duration.between(start, end).toMinutes()` with `Instant` or `ZonedDateTime` | With `LocalDateTime` it computes wall-clock difference — wrong across DST |

**Daylight saving time in Tallinn.** Estonia uses EET (UTC+2) in winter and EEST (UTC+3) in summer.

| When (2026) | What happens at 03:00 / 04:00 local time | Effect |
|---|---|---|
| Last Sunday of March — **29 March** | at 03:00 clocks jump to 04:00 | 03:00–03:59 does not exist; the day has 23 hours |
| Last Sunday of October — **25 October** | at 04:00 clocks go back to 03:00 | 03:00–03:59 happens **twice**; the day has 25 hours |

Now imagine a night shift logged in March from **02:30 to 04:30** local time:

```
Wall clock:   02:30 ──────────── 04:30          naive difference: 2 h 00 min  ✗
Real time:    02:30 → 03:00 (30 min), clock jumps to 04:00, 04:00 → 04:30 (30 min)
              = 1 h 00 min  ✓   (in UTC: 00:30Z → 01:30Z)
```

If the code subtracts wall-clock times, the client pays for an hour that never happened. In October the
error goes the other way (a 3-hour entry shows as 2 hours), and a time like "03:30 on 25 October" is
**ambiguous** — it happened twice. PHP resolves it to the second occurrence (+02:00), Java's
`atZone()` to the first (+03:00). The real fix is to store instants (UTC, or a timestamp *with* time
zone) and convert to local time only for display. Your intermediate task explores what *your* app does.

**ISO weeks** (the weekly report). ISO 8601 weeks start on Monday, and week 1 is the week that contains
the year's first Thursday. So a date can belong to a week of *another* year:

| Date | ISO week |
|---|---|
| Thu 31 Dec 2026 | 2026-W53 |
| Fri 1 Jan 2027 | **2026**-W53 |
| Mon 4 Jan 2027 | 2027-W01 |

PHP: `$date->format('o-\WW')` — `o` is the ISO year; `Y` would give "2027-W53", which is wrong.
Java: `date.get(IsoFields.WEEK_BASED_YEAR)` and `date.get(IsoFields.WEEK_OF_WEEK_BASED_YEAR)`.

**Adding days and skipping weekends (R12).** *Due date = issue date + 14 days; if that is a Saturday or
Sunday, move to the next Monday.*

```
due = issuedOn + 14 days
if due is Saturday → due + 2 days
if due is Sunday   → due + 1 day
```

Think before you code: 14 days is exactly two weeks, so the due date is always the **same weekday** as
the issue date. The weekend rule only triggers for invoices issued on a weekend. Noticing this helps you
choose test cases (issue on Friday, Saturday, Sunday, Monday; across a month end; across a year end).
Public holidays are not in R12 — do not add them silently; ask.

**Calendar days vs 24 hours.** "Plus one day" and "plus 24 hours" are different on DST days:
`plusDays(1)` on a `ZonedDateTime` keeps 09:00 → 09:00; `plusHours(24)` moves 09:00 → 08:00 or 10:00.
Use dates (`LocalDate`, Carbon `startOfDay()`) for due dates, instants for durations.

### 9. Boolean logic

**Truth tables** list every combination of inputs:

| a | b | a AND b | a OR b | NOT a | a XOR b |
|---|---|---|---|---|---|
| false | false | false | false | true | false |
| false | true | false | true | true | true |
| true | false | false | true | false | true |
| true | true | true | true | false | false |

**De Morgan's laws** let you remove a negation from a whole expression:

```
NOT (a AND b)  ==  (NOT a) OR (NOT b)
NOT (a OR b)   ==  (NOT a) AND (NOT b)
```

Example: "skip the warning unless it is capped and not yet warned":

```php
if (! ($project->isCapped() && $project->budget_warning_sent_at === null)) { return; }
// De Morgan → easier to read:
if (! $project->isCapped() || $project->budget_warning_sent_at !== null) { return; }
```

**Short-circuit evaluation.** `&&` stops at the first `false`, `||` stops at the first `true`. The
order of conditions therefore matters:

```php
$project !== null && $project->isCapped()     // safe: isCapped() is not called on null
$project->isCapped() && $project !== null     // crashes when $project is null
```

In Java, `&` and `|` on booleans do **not** short-circuit — use `&&` and `||`.

**Guard clauses** are the practical result: check each "nothing to do" condition at the top of a
method and `return` early. `BudgetMonitor::check()` from lesson 09 does exactly this. Compare:

```
Nested                                    Guard clauses
if capped:                                if not capped: return
    if not warned:                        if already warned: return
        if value >= 80 %:                 if value < 80 %: return
            notify                        notify
            set timestamp                 set timestamp
```

Same logic, but the right side reads top to bottom, and each rule is one line.

**Decision tables.** When several conditions combine, write every combination as a table *before*
coding. Here is R9 as a decision table:

| # | Capped? | Already warned? | Value ≥ 80 % of cap? | → Send warning? |
|---|---|---|---|---|
| 1 | no | – | – | no |
| 2 | yes | yes | – | no |
| 3 | yes | no | no | no |
| 4 | yes | no | yes | **yes** |

`–` means "does not matter" — the answer is the same for both values, so two rows collapse into one.
Three yes/no conditions would give 2³ = 8 rows; the don't-care entries reduce them to 4. Check that the
table is **complete** (every combination is covered by exactly one row) and **consistent** (no two rows
say different things for the same input).

A decision table gives you three things at once:

1. **The code.** Each row becomes a guard clause, or one arm of a `match`/`switch` expression.
2. **The tests.** Each row is one test case (lesson 11: a data provider / `@CsvSource`, one row per
   table row).
3. **A document** a non-programmer can check: "Is row 2 really what you want?"

Your basic independent task is the decision table for invoice actions (R13), which lesson 13 turns
into code.

### 10. Value objects and immutability

In lesson 09 we named **primitive obsession**: using `int` for money. What is wrong with `int $amount`?

- Is it cents or euros? `hourly_rate_cents` says it in the name; a method parameter `int $amount` does
  not.
- You can pass minutes where cents are expected, and nothing complains.
- Rounding and formatting code is copied to every place that needs it.

A **value object** fixes this. It is a small class that:

| Property | Meaning | In `Money` |
|---|---|---|
| Represents a value, not an identity | Two objects with the same data are equal | `Money::fromCents(500)` equals another `Money::fromCents(500)` |
| **Immutable** | Never changes after creation | `add()` returns a *new* `Money` |
| Valid by construction | Invalid values cannot exist | `fromEuros("85.505")` throws; overflow throws |
| Has behaviour | Operations that make sense for the value | `add`, `subtract`, `percentageBp`, `format` |

Why immutability matters:

```php
$rate = Money::fromCents($project->hourly_rate_cents);
$total = $rate->add($bonus);        // immutable: $rate is unchanged, $total is new
// If add() changed $rate itself, every other place holding $rate would suddenly see a
// different hourly rate — a bug that shows up far away from its cause.
```

PHP 8.2+ has `final readonly class`: every property can be set only once, in the constructor. Java
has `record`: all fields are `final`, and `equals`, `hashCode` and `toString` are generated. Both make
immutability the default instead of a discipline.

**Why static factory methods** (`Money::fromCents(8550)`, `Money::fromEuros('85.50')`) instead of
`new Money(...)`? The name says which unit you pass. `new Money(8550)` is ambiguous; `fromEuros` and
`fromCents` are not.

**Formatting choice.** Our `format()` returns Estonian style `1 234,50 €`: a space between thousands,
a comma before the cents, the euro sign after the number with a space. Both frameworks have currency
formatters — Laravel's `Number::currency()` and Java's `NumberFormat.getCurrencyInstance()`, which
lesson 04 used. `Money` still formats by itself: it is exact (integer cents, no float on the way), and
it uses a normal space, so tests can compare with plain strings. Real locale formatting (ICU /
`NumberFormatter` / `java.text.NumberFormat`) uses a *non-breaking* space (U+00A0), so the amount never
breaks across two lines. The advanced task explores this.

---

## Laravel vs Spring Boot

| Idea | Laravel / PHP | Spring Boot / Java |
|---|---|---|
| Integer type for cents | `int` (64-bit) | `long` |
| Overflow behaviour | Silently becomes `float` | Silently wraps; use `Math.addExact` etc. |
| Integer division | `intdiv($a, $b)` (`/` gives a float!) | `a / b` with integer operands |
| Floor division / modulo | `(int) floor($a / $b)` | `Math.floorDiv`, `Math.floorMod` |
| Value object | `final readonly class Money` | `record Money(long cents)` |
| Decimal type | `BcMath\Number` (PHP 8.4) or `bcmath` strings + `bcround()` | `BigDecimal` + `RoundingMode` |
| Date without time | `CarbonImmutable` + `startOfDay()` | `LocalDate` |
| Instant | Carbon with a time zone, or `getTimestamp()` | `Instant`, `ZonedDateTime` |
| ISO week | `format('o-\WW')` | `IsoFields.WEEK_BASED_YEAR`, `WEEK_OF_WEEK_BASED_YEAR` |
| Locale formatting | `NumberFormatter` (intl), `Number::currency()` | `NumberFormat.getCurrencyInstance(locale)` |

---

## Common mistakes

| Mistake | Why it hurts | What to do instead |
|---|---|---|
| `float`/`double` for money, "just for this one division" | 19,99 € becomes 1998 cents; results differ at boundaries | Integer cents, integer half-up division |
| `(int)` cast or `intdiv` where you meant rounding | Truncation silently rounds toward zero | Say which mode you want: `(n + d/2) div d` for half up |
| Rounding each entry's amount, then adding | The total is off by cents and depends on the number of entries | Round minutes per entry (R5), money once per line (R3) |
| VAT per line instead of on the subtotal | Sum of line VATs ≠ VAT of the subtotal | VAT once on the invoice subtotal |
| `$percent = $value / $cap * 100` then `>= 80` | Float comparison at the exact boundary | `value × 100 >= cap × 80` |
| Rounding the threshold instead of comparing exactly | With cap 10,04 €, 80 % is 8,032 € → rounds to 8,03 €, and a value of 8,03 € would already warn | Compare `value × 100 >= cap × 80` — no rounding at all |
| Java `Math.round()` on negative halves | `Math.round(-2.5)` is −2 | Explicit `RoundingMode` or our `divideHalfUp` |
| Subtracting wall-clock times | 1 h across the March DST change becomes 2 h | Subtract instants / timestamps |
| `format('Y-\WW')` for ISO weeks | 1 Jan 2027 is shown as week 53 of 2027 | `o` (PHP) / `WEEK_BASED_YEAR` (Java) |
| Deeply nested `if`s for combined conditions | Missing and contradicting cases hide inside | Decision table → guard clauses or `match` |
| Mutable money objects | A change in one place shows up somewhere else | Immutable value objects |

---

## Independent work (~2 h)

Do these at home after the walkthrough, in your own project (~2 h). Tasks build on each other across lessons, so finish at least the **Basic** ones. Solutions are at the bottom of the [Laravel](laravel.md#independent-work--solutions) and [Spring Boot](spring-boot.md#independent-work--solutions) walkthroughs — try first, compare after.

**Basic** tasks are required so your project stays in line with [PROJECT.md](../../PROJECT.md).

### Task 1 — `DueDateCalculator` (Basic)

Implement R12 in `DueDateCalculator` (Laravel: `App\Support\DueDateCalculator`; Spring:
`common.DueDateCalculator`) with one public method that takes the issue date and returns the due date.

Acceptance criteria:

- [ ] Works with dates only (no time of day): Laravel `CarbonImmutable`, Java `LocalDate`.
- [ ] 14 is a named constant, not a magic number.
- [ ] You checked by hand (Tinker / jshell) and wrote down the results for: issued Thursday
      1 Oct 2026, Saturday 3 Oct 2026, Sunday 4 Oct 2026, and Saturday 19 Dec 2026 (crosses the year).
- [ ] No database, no `now()` inside the class — the caller passes the date.

### Task 2 — Decision table: invoice actions (Basic)

Invoices arrive in lesson 13, but the rules exist already (R13, R14 and "draft → sent → paid").
Create `docs/invoice-rules.md` with:

- [ ] A decision table with the statuses `draft`, `sent`, `paid` as rows and the actions **edit**,
      **delete**, **send**, **mark as paid** as columns; every cell is *allowed* or *not allowed*.
- [ ] A state diagram (Mermaid `stateDiagram-v2`) of the allowed transitions.
- [ ] Pseudocode of one function `isAllowed(status, action): bool` that implements the table.
- [ ] One sentence per "not allowed" cell that is not obvious, explaining *why* (for example, why a
      sent invoice cannot be deleted).
- [ ] A note that lesson 13 implements this with the `InvoiceStatus` enum and that each cell becomes a
      test.

### Task 3 — DST experiment (Intermediate)

Find out whether *your* Billable calculates durations correctly on the day the clocks change.

- [ ] In Tinker / jshell, compute the minutes between 29 March 2026 02:30 and 04:30 in
      `Europe/Tallinn` in two ways: naive wall-clock difference and instant-based difference. Record
      both results.
- [ ] Log that entry through your time sheet form. Record what the time sheet and the project page
      show.
- [ ] If the app shows 120 minutes, fix it so it shows 60. Write two or three sentences on what caused
      it and what the October case (25 October 2026, 03:00–03:59 happens twice) means for your storage
      choice.

### Task 4 — Locale-aware formatting (Advanced)

Add a method to `Money` that formats for a given locale using the platform's formatter (PHP
`NumberFormatter` from the `intl` extension; Java `java.text.NumberFormat`).

- [ ] `et_EE` / `et-EE` gives `1 234,50 €`, and `en_IE` / `en-IE` gives `€1,234.50`, for 123 450 cents;
      negative amounts work.
- [ ] You explain in a doc comment why the output contains a non-breaking space (U+00A0) and how this
      affects tests.
- [ ] Java: no `double` is created on the way. PHP: explain in a comment why passing a float to
      `NumberFormatter` is acceptable *only* for display, after all calculation is finished.

---

## Self-check

1. Why is `(int) (19.99 * 100)` equal to 1998?
2. Round −2.5 with half up, half even, ceiling and floor.
3. Three entries of 10 minutes at 85 €/h: what is the difference between rounding per line and rounding
   once, and which does Billable do?
4. Calculate the VAT on a subtotal of 100,01 € at 24 % using integer arithmetic.
5. Round 16 minutes up to a 15-minute increment using only integer operations. Show the formula.
6. Why can a 2-hour wall-clock entry on 29 March in Tallinn really be 1 hour?
7. Write `NOT (isDraft AND hasNoPayments)` without the outer `NOT`.
8. Name three properties of a value object.

<details>
<summary>Answers</summary>

1. `19.99` cannot be stored exactly in binary; the product is 1998.9999999999998, and the `(int)` cast
   cuts off the decimals (truncation toward zero).
2. Half up: −3. Half even: −2. Ceiling: −2. Floor: −3.
3. Per line: 3 × 1 417 = 4 251 cents. Once: 30 min × 8 500 ÷ 60 = 4 250 cents. Billable rounds
   *minutes* per entry but *money* once per project line, so it gives 42,50 €.
4. 10 001 × 2 400 = 24 002 400; + 5 000 = 24 007 400; div 10 000 = **2 400 cents = 24,00 €** (exact value
   2 400.24).
5. `(16 + 15 − 1) div 15 × 15 = 30 div 15 × 15 = 2 × 15 = 30`.
6. At 03:00 the clocks jump to 04:00, so the hour 03:00–03:59 does not exist. From 02:30 to 04:30 on the
   wall clock only 60 real minutes pass.
7. `NOT isDraft OR NOT hasNoPayments` — i.e. "not a draft, or has payments" (De Morgan).
8. Any three: compared by value, immutable, valid by construction, carries behaviour for the value.

</details>

---

## Checklist

- [ ] No `float`/`double` anywhere money is calculated (search your code for `float`, `double`, `/ 100`)
- [ ] `Money` exists, is immutable, parses euros without floats and throws on overflow
- [ ] `TimeRounding::roundUp()` uses ceiling division and rejects invalid increments
- [ ] Billing strategies take and return `Money`; VAT and the hourly amount round half up with the same
      function (R3, R6)
- [ ] The project page shows billable so far, VAT 24 % and total — calculated per-entry-rounded
- [ ] Project page and `BudgetMonitor` use the **same** billable-minutes method
- [ ] `DueDateCalculator` implements R12; `docs/invoice-rules.md` has the decision table
- [ ] You can explain every rounding step in your code (ÕV2) — try it aloud once
- [ ] Formatter passes and commits are small and clear (ÕV7)

---

## Further reading

- PHP: [Floating point numbers](https://www.php.net/manual/en/language.types.float.php) · [intdiv](https://www.php.net/manual/en/function.intdiv.php) · [NumberFormatter](https://www.php.net/manual/en/class.numberformatter.php) · [DateTime::format](https://www.php.net/manual/en/datetime.format.php)
- Carbon: [Documentation](https://carbon.nesbot.com/docs/)
- Java: [RoundingMode](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/math/RoundingMode.html) · [BigDecimal](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/math/BigDecimal.html) · [Math](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/Math.html) · [java.time](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/time/package-summary.html) · [NumberFormat](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/text/NumberFormat.html)
- Java: [Records](https://docs.oracle.com/en/java/javase/25/language/records.html)
- PHP: [Readonly classes](https://www.php.net/manual/en/language.oop5.basic.php#language.oop5.basic.class.readonly)
