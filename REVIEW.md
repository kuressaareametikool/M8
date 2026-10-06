# Course review — 2026-10-01

Focus: hand-rolled code where framework/stdlib has built-in; outdated APIs for Laravel 12 / PHP 8.4 / Boot 4 / Java 25; factual errors; track drift.
Severity: H = wrong/won't run, M = should fix, L = nice to have. Line numbers = file state at review time.

## Status — 2026-10-01

All High, Medium and Low items applied (fixed, or a short note naming the built-in where the full fix would ripple). Backup of the pre-fix state: session scratchpad `ta-25-backup`.

Decisions taken:
- Lesson 04 `euros()` / `Formatters.euros` now use `Number::currency` / `NumberFormat.getCurrencyInstance` (NBSP output; needs ext-intl). Lesson 10 `Money::format()` keeps its own exact formatting with normal spaces.
- Java `Money.sum(Money...)` varargs in both tracks.
- Laravel guard: `Model::preventSilentlyDiscardingAttributes()` from lesson 03; `preventLazyLoading()` still introduced in lesson 06.

Cross-lesson follow-ups — ✅ all 10 done 2026-10-01 (backup before this pass: scratchpad `ta-25-backup-2-after-fixes`). Open choices: Spring flash box hidden on error pages via `timestamp == null` (flash key `status` collides with Boot's error model — renaming the flash key would be cleaner); Spring release-on-failure is redundant inside REQUIRES_NEW (kept for parity, text says so).

Items:
1. Spring 404 vs 500 for unknown project (`TimeEntryService` → `NotFoundException`; lesson 12 tests expect `IllegalArgumentException`).
2. Atomic "send budget warning once" (09 both tracks; lesson 12 BudgetMonitor tests change).
3. Spring `@ConfigurationProperties` record for `billable.*` (09, 10, 13, 15).
4. Spring CSRF via `spring-boot-starter-security` + `permitAll()` (05 onward; every MockMvc POST needs `.with(csrf())`).
5. Spring one-indexed pagination (all Spring pagination links).
6. Testcontainers `@ServiceConnection` from lesson 01 instead of 14 (01, 02 CI, 14).
7. Thymeleaf parameterised layout fragment (every Spring template).
8. Laravel NOT NULL timestamps to match Flyway (every migration; PROJECT.md says schemas identical).
9. `Model::shouldBeStrict()` (adds `preventAccessingMissingAttributes`) in lesson 06.
10. Laravel `#[Config]` contextual attribute for `MailNotifier` (09 + lesson 12 rewrite).

## Cross-cutting themes

1. **Money formatting hand-rolled in both tracks** — `euros()` (04 L), `Formatters.euros` (04 S), `Money::format()` regex (10 L/S), `cents_to_euros_input`/`toEuros` (05). Built-ins: `Number::currency($c / 100, in: 'EUR', locale: 'et')` (ext-intl), `NumberFormat.getCurrencyInstance(Locale.of("et","EE")).format(BigDecimal.valueOf(cents, 2))`. Pedagogy (integer cents, R1) can stay — but name the built-in each time, and fix the README 04:294 claim that floats-for-display break R1.
2. **Eloquent strict mode missing** — add `Model::shouldBeStrict(! app()->isProduction())` in lesson 03 (fixes 03 README fillable error, 03 troubleshooting row, N+1 parity with Spring `open-in-view=false`).
3. **Enum converter hand-written (Spring 03, 13)** — Jakarta Persistence 3.2 / Hibernate 7: `@EnumeratedValue` on `dbValue` + `@Enumerated(STRING)`; drop `AttributeConverter`.
4. **Testcontainers late** — Spring 01/02 work around Compose-skipped-in-tests with CI `services: postgres`; adopt `@ServiceConnection` from lesson 01 (lesson 14 uses it anyway).
5. **Query counting by hand** — `DB::listen` (06, 14) → `$this->expectsDatabaseQueryCount($n)`.
6. **Missing factories** — no `TimeEntryFactory` / `InvoiceFactory`; invoice arrays copy-pasted 3×. Use factories + states + `sequence()` + `recycle()`.
7. **Track inconsistencies** — page indexing (0 vs 1 → `spring.data.web.pageable.one-indexed-parameters=true`), CSRF on/off, nullable vs NOT NULL timestamps, 404 vs 500 for missing project, `Money.sum` list vs variadic, Spring invoice list unpaginated.

## High — ✅ all fixed 2026-10-01 (plus `15/laravel.md:1057` diagram and `07/laravel.md` tinker archive demo)

- `04-mvc-reading-data/README.md:150` — claims Laravel `/clients/abc` → 404. On PostgreSQL bigint bind fails (SQLSTATE 22P02) → 500. Fix: `Route::pattern('client', '[0-9]+')` (and `project`) or `->where(...)`, then fix table.
- `06-orm-relationships/spring-boot.md:1619-1622` — `INSERT INTO time_entries` omits NOT NULL `created_at`/`updated_at` → fails. Add columns, select `now(), now()`.
- `03-orm-basics/README.md:296-297` — "anything else throws an exception". Wrong: Eloquent silently drops non-fillable keys unless `preventSilentlyDiscardingAttributes()`. Fix text + recommend `shouldBeStrict()`.
- `03-orm-basics/laravel.md:852` — troubleshooting row for "Add [archived_at] to fillable" can't occur (model has `$fillable` → silent drop). Same fix.
- `03-orm-basics/spring-boot.md:167-231` — says `@Enumerated(STRING)` only stores `HOURLY`; outdated since JPA 3.2. Use `@EnumeratedValue`. Also README:398, recap :1017, and `13/spring-boot.md:119-183, :1926`.
- `10-math-and-logic/README.md:125-126` — "bcmath truncates, add rounding yourself". PHP 8.4 has `bcround()/bcfloor()/bcceil()` + `BcMath\Number`. Update also :100, :479.
- `09-patterns-events-and-adapters/laravel.md:1007-1009` — "cannot let container auto-build LoggingNotifier… build with `new`". Laravel has `$this->app->extend(Notifier::class, fn ($inner, $app) => new LoggingNotifier($inner, ...))` — native decorator support.

## Medium

### Laravel
- `04/laravel.md:151-206` — `euros()` + composer `files` → `Number::currency`, via accessor or Blade.
- `05/laravel.md:709-735` — `euros_to_cents`/`cents_to_euros_input`; name PHP 8.4 `BcMath\Number` / brick/money. Bug: `cents_to_euros_input(-5)` → `"0.-5"`.
- `05/laravel.md:773-816` — manual nulling → `exclude_unless:billing_type,...`; `$this->enum('billing_type', BillingType::class)`, `$this->integer(...)`.
- `06/laravel.md:1102` — `(string) $request->query('week')` with `?week[]=x` → 500. Validate with regex rule.
- `06/laravel.md:1319-1329` + `06/README.md:397` — `DB::listen` counter → `expectsDatabaseQueryCount()`.
- `03/laravel.md:763-765` — no lazy-loading guard; add `preventLazyLoading`/`shouldBeStrict` here.
- `03/laravel.md:1064-1091` + `03/README.md:477` — "raw SQL better" example is `withCount`/`withSum`. Pick one ORM can't do (`generate_series`, window fn).
- `09/laravel.md:319-367` — "container cannot guess a string" outdated → `#[Config('billable.owner_email')]` or `->giveConfig()`.
- `09/laravel.md:257-312` — never names Laravel Notifications (`Notification::route(...)->notify()`, `Notification::fake()`); add one sentence why course hand-rolls.
- `09/README.md:265-267` — "mocking framework mail API is hard" misleading given `Mail::fake()`. Reframe around owning the abstraction.
- `07/README.md:255-260, :421-427` — missing `scoped()` and `#[Singleton]`/`#[Scoped]`/`#[Bind]` attributes.
- `10/laravel.md:185-191` — regex thousands grouping → `number_format(intdiv(...), 0, ',', ' ')`.
- `10/laravel.md:539-540` (+ `spring-boot.md:521`) — second half-up rounding impl duplicates `Money::divideHalfUp`; make it public, reuse.
- `10/README.md:178` (+ L :531/:625, S :515/:618) — "only place a fraction is rounded" false (VAT `percentageBp` too).
- `11/laravel.md:1451-1452` — docs example says format >1000 € untested; contradicts test at :374.
- `13/laravel.md:947-951, :30, :2179` — `array_reduce` subtotal; use `Money::sum()` built in lesson 11.
- `13/laravel.md:1685, :1859`, `14/laravel.md:350-361` — no `TimeEntryFactory`.
- `13/laravel.md:264, :1976`, `14/laravel.md:976, :1124` — `HasFactory` on `Invoice` but no factory; arrays copy-pasted.
- `14/README.md:176-182` — optimistic lock on `updated_at` (`timestamp(0)`, 1-second resolution) → same-second updates both pass. Use integer `version`.
- `14/laravel.md:363-373, :1135-1144` — `DB::listen`/`enableQueryLog` → `expectsDatabaseQueryCount()`.
- `15/laravel.md:1057` — state diagram `draft --> draft : edit`; no edit action exists, Spring diagram lacks it. Remove.
- `15/laravel.md:1357-1366, :190` — `pg_isready` loop → `docker compose up -d --wait` (healthcheck exists).

### Spring
- `04/spring-boot.md:235-269` — `Formatters.euros` → `NumberFormat.getCurrencyInstance(Locale.of("et","EE"))`.
- `04/README.md:294-295` — "formatting needs integer division or breaks R1" — not true of built-ins (`BigDecimal.valueOf(cents, 2)` exact).
- `05/spring-boot.md:739-808` — claims annotation "cannot express" conditional rules, manual `checkBillingRules`; lesson 06 uses `@AssertTrue` for same thing. Use `@AssertTrue` or `Validator` + `@InitBinder`.
- `05/README.md:270-271`, `05/spring-boot.md:641-644` — "no login → no CSRF risk" wrong for intranet app; Laravel keeps CSRF. Reword or add Security with `permitAll()` + CSRF.
- `06/spring-boot.md:695` — Boot warns about open-in-view only when property unset; here it's set explicitly.
- `06/spring-boot.md:1323-1341` — hand ISO-week parsing → `LocalDate.parse(week + "-1", DateTimeFormatter.ISO_WEEK_DATE)`.
- `03/spring-boot.md:1220-1267` — "raw SQL better" example doable in JPQL with DTO projection.
- `01/spring-boot.md:263-267, :568-576`, `02/spring-boot.md:394-407` — Testcontainers `@ServiceConnection` instead of compose-running-for-tests / CI services block.
- `07/spring-boot.md:892` — final `@Transactional` *method* doesn't give CGLIB error (only final class does); method silently runs untransactional. Split row.
- `09/spring-boot.md:85-100, :279-286` — scattered `@Value` + free-string notifier type → `@ConfigurationProperties` record + enum + `@Validated`.
- `10/spring-boot.md:337` — `roundUp` formula → `Math.ceilDiv(minutes, increment) * increment`.
- `10/spring-boot.md:178-182` — regex grouping → `DecimalFormat` with symbols.
- `10/spring-boot.md:1138` — Javadoc says write test string with normal spaces; contradicts NBSP note before it.
- `11/spring-boot.md:1286-1323` — PIT targets exclude `budget.*` but example discusses `BudgetMonitor` mutant.
- `13/spring-boot.md:1077-1078, :2180` — `reduce(fromCents(0), Money::add)` → `Money.sum(...)` (lesson 11 says it was built for this).
- `13/spring-boot.md:1383-1389` — `@PastOrPresent` uses JVM clock, not `Clock` bean → fixed-clock tests depend on real date. Wire `ClockProvider` via `ValidationConfigurationCustomizer`.
- `14/spring-boot.md:274, :398-401, :1088, :1272` — Boot 3.4+ `@AutoConfigureTestDatabase` defaults `Replace.NON_TEST`; "would replace with embedded DB" + H2 row outdated.

## Low

- `01/spring-boot.md:359-468` — repeated head/nav/flash fragments → parameterised layout fragment `~{layout :: layout(~{::title}, ~{::main})}`.
- `01/spring-boot.md:837-841` — `th:attr="aria-current=..."` → `th:aria-current`.
- `01/spring-boot.md:896` — `/ 1_000_000` → `Duration.ofNanos(...).toMillis()`; mention Actuator `http.server.requests`.
- `01/README.md:348` — `spring-boot-starter-web` is deprecated alias in Boot 4, not "does not compile".
- `01/README.md:337` — mention component layouts (`<x-layout>`) as Laravel 12 default style.
- `02/laravel.md:668` — pre-commit hook → `pint --test --dirty`.
- `03/spring-boot.md:719` — `LocalDateTime.now()` default zone; use Tallinn zone / `Clock`.
- `03/spring-boot.md:293-299` — `@CreationTimestamp` ignores `Clock`; mention JPA auditing + `DateTimeProvider`.
- `03/spring-boot.md:1254`, `03/laravel.md:1078` — LIKE search doesn't escape `%`/`_`.
- `03/spring-boot.md:63-68` — SQL TRACE logging in shared props → `application-dev.properties`.
- `03/laravel.md:152, :226` — `timestamps()` nullable vs Flyway NOT NULL; PROJECT.md says identical schemas.
- `04/laravel.md:339-343, :564` — `Paginator::defaultView('partials.pagination')` once.
- `04/laravel.md:566` — `optional($x)?->` redundant.
- `04/spring-boot.md:183-194` — 3 queries vs `withCount`; JPQL `count` projection or `@Formula`.
- `04/spring-boot.md:205` — page index 0 vs 1 between tracks → `one-indexed-parameters=true`.
- `06/laravel.md:800-806` — `strtotime` → `$this->date(...)->diffInSeconds(...)`.
- `06/laravel.md:856-862` — rebuilding array → `TimeEntry::create($request->validated())`.
- `06/laravel.md:1211` — `Rule::when(...)`.
- `06/README.md:259` — mention `automaticallyEagerLoadRelationships()` (12.8+) as aside.
- `06/spring-boot.md:1452-1462` — hand-rolled filter validation → `@Valid` record + `BindingResult`.
- `07/spring-boot.md:280-319` — Spring throws `IllegalArgumentException` (500) vs Laravel 404.
- `08/laravel.md:334-336` — mention `#[Tag('billing')] iterable $strategies`.
- `08/laravel.md:775-843` — mention custom Eloquent builder (`#[UseEloquentBuilder]`).
- `08/spring-boot.md:324-343` — `Collectors.toMap(..., EnumMap::new)` (default merge throws on dup).
- `08/README.md:334` — `Sort.by()` is static factory, not Builder.
- `09/laravel.md:693-699` — try/catch+report → `rescue(..., report: true)`.
- `09/laravel.md:600-630`, `09/spring-boot.md:555-590` — "send once" is check-then-set race; conditional `UPDATE ... WHERE sent_at IS NULL`.
- `09/README.md:175` — "job retried later" untrue for Spring `@Async`.
- `09/README.md:213-224` — point at Spring Modulith `@ApplicationModuleListener`.
- `10/README.md:287-289` — name `Math.ceilDiv`.
- `10/README.md:149-150` — PHP 8.4 `RoundingMode` enum.
- `10/README.md:110` — `PHP_INT_MAX + 1` output differs across files; use `9.223372036854776E+18`.
- `10/laravel.md:964-973`, `10/spring-boot.md:965-971` — mention Carbon `isWeekend()`/`next(MONDAY)`, `TemporalAdjusters.next`.
- `10/laravel.md:1095-1104` — assumes timezone still UTC; lesson 01 already sets Tallinn.
- `10/laravel.md:55` — garbled sentence about `/`.
- `11/README.md:479` — mention PHPUnit `#[TestWith]`.
- `11/spring-boot.md:855` vs `11/laravel.md:969` — `sum(List)` vs variadic.
- `12/laravel.md:764` — example class `HourlyBilling` isn't final; use `Money`.
- `13/laravel.md:2127-2128` — test sums input, not `allocate()` output.
- `13/laravel.md:2244`, `13/spring-boot.md:2228` — `Benchmark::measure()` / JMH.
- `13/spring-boot.md:1147-1150` — since Spring 6.0 non-private methods proxied; "only public" outdated.
- `13/spring-boot.md:1220` — invoice list unpaginated vs Laravel `paginate(20)`.
- `14/README.md:233-237` — mention `createOrFirst()`.
- `14/laravel.md:656-667` — atomic `INSERT ... ON CONFLICT DO UPDATE ... RETURNING`.
- `14/spring-boot.md:136-212` — `JdbcTestUtils` / `@Sql`; `JdbcClient` over `JdbcTemplate`.
- `15/laravel.md:1379`, `15/spring-boot.md:1400` — `curl --retry-connrefused` instead of sleep.
- `15/spring-boot.md:201` — `postgres:17` vs `postgres:17-alpine`.
- `15/spring-boot.md:958` — `version` "editing drafts" — drafts aren't editable.
- `15/README.md:244-247` — Java 23+ Markdown doc comments (`///`).
