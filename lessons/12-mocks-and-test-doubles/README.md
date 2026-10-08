# 12 — Mocks and test doubles

**Outcomes:** ÕV6 — *kasutab testides mock-klasse* · **Time:** 2 h in class + ~2 h independent
**Before this lesson:** your project matches the end state of lesson 11 (see [what exists after each lesson](../../PROJECT.md#what-exists-after-each-lesson))
**Walkthroughs:** [Laravel](laravel.md) · [Spring Boot](spring-boot.md)

---

## Why this matters

In lesson 11 you tested pure code: numbers in, numbers out. The rest of Billable is not like that.
`BudgetMonitor` reads a project, adds up time entries, **sends an email** and writes a timestamp.
`TimeEntryService` saves an entry, asks **what time it is** and **publishes an event**.

How do you test "the owner gets exactly one email when a capped project reaches 80 %"?

- With the real mailer, every test run sends an email. Tests become slow, depend on a mail server, and
  you cannot easily check *how many* emails were sent.
- With the real clock, the rule "an entry cannot end in the future" gives a different result every
  second. A test written today fails tomorrow.
- With a real external API (a currency rate, a payment provider), tests fail when the network is down
  and may even cost money.

The answer is a **test double**: an object that stands in for a real collaborator during a test, like a
stunt double stands in for an actor. The test decides how the double behaves and can ask it afterwards
what happened.

This is where lessons 07 and 09 pay off. Because `BudgetMonitor` receives its `Notifier` through the
constructor, a test can hand it a double. Because the notifier is **our** interface (the Adapter from
lesson 09), the double is tiny. Code that creates its collaborators itself (`new MailNotifier()`,
`now()` deep inside a method) cannot be tested this way.

In the defence you will be asked: *"This test uses a mock. What would break if you used the real
class?"* This lesson gives you the words and the reasons.

---

## Today's goal

At the end of class your project has:

- tests for the budget warning (R9): sent at 80 %, **not** at 79 %, **not** a second time, **not** for
  hourly projects — with the notifier replaced by a mock;
- the clock under control in those tests, so the stored `budget_warning_sent_at` is an exact, known
  value;
- a test that `TimeEntryService` publishes the `TimeEntryLogged` event, and does not publish it when the
  entry is rejected;
- a hand-written fake notifier (`InMemoryNotifier`) used in one test, compared with the mock version;
- Laravel: a `Mail::fake()` test for `MailNotifier`. Spring: a `@WebMvcTest` for `ClientController`.

By the end you can:

- name the five kinds of test doubles and give a Billable example of each;
- explain the difference between checking **state** and checking **interactions**;
- write a test that proves something did **not** happen;
- control the current time in a test;
- explain "don't mock what you don't own" and why Billable has a `Notifier` interface;
- recognise an over-mocked test and say when a fake is the better choice.

---

## Concepts

### 1. Why test doubles?

A unit test should be fast, isolated and repeatable (lesson 11, FIRS). Some collaborators break those
properties:

| Collaborator | Problem in a test | Billable example |
|---|---|---|
| Mail server | Slow, needs a running server, side effect that cannot be undone | `MailNotifier` → Mailpit |
| Clock | Changes every second → results change | "entry ends in the future", `budget_warning_sent_at` |
| External API | Network, rate limits, cost, unpredictable answers | Currency rates (a capstone extension) |
| Database | Slower, needs setup and cleanup, shared state between tests | `ProjectRepository`, Eloquent queries |
| Randomness | Different every run | Generated tokens, shuffled data |

A test double replaces such a collaborator so that the test controls it completely.

**Not every collaborator needs a double.** `Money`, `TimeRounding` and `HourlyBilling` are pure and
fast. Use the real ones. Replace only what is slow, unpredictable or has side effects.

### 2. The five kinds of test doubles

Gerard Meszaros named five kinds of doubles in *xUnit Test Patterns*. People often say "mock" for all
of them, but the difference matters when you explain a test.

| Kind | What it does | Billable example |
|---|---|---|
| **Dummy** | Is passed in only because a signature needs it; never used | A `Client` object that `ProjectTestData` passes to the `Project` constructor; a `Notifier` handed to `BudgetMonitor` in a test for an hourly project (it returns before notifying) |
| **Stub** | Returns answers the test prepared ("canned answers"); no checks | `ProjectRepository.findById(7)` returns our test project; a fixed clock that always says "5 Oct 2026, 14:30" |
| **Spy** | Records how it was called; the test checks the record **after** the act | A Mockery spy / Mockito `verify` on a notifier after `check()` ran |
| **Mock** | Has **expectations** set before the act; fails if they are not met | A notifier that must receive `notify(...)` exactly **once** |
| **Fake** | A real, working, simplified implementation | `InMemoryNotifier` that collects messages in a list; SQLite in memory instead of PostgreSQL; `Mail::fake()` |

The practical difference to remember: a **stub answers questions** (it feeds data *into* the code under
test), a **mock or spy checks calls** (it observes what comes *out*). Mockito and Mockery can create all
of these; the name describes how you *use* the object, not which library made it.

```
                  ┌─────────────────────────┐
  stub  ─────────►│                         │─────────► mock / spy
  (input:         │   code under test       │   (output: "was notify()
  "project 7 is   │   BudgetMonitor         │    called once, with
   capped, 80 %") │                         │    this subject?")
  fake clock ────►│                         │
                  └─────────────────────────┘
```

### 3. State verification vs interaction verification

There are two ways to check that code did the right thing:

- **State verification:** after the act, look at the *result* or at the *state* of objects.
  "`budget_warning_sent_at` is now 2026-10-05 14:30." "The returned amount is 90.00 €."
- **Interaction verification:** check that the code *called* a collaborator in a certain way.
  "`notify()` was called once, and the subject contains the project name."

Prefer state verification when you can: it tests *what* happened, not *how*. Use interaction
verification when the interaction **is** the behaviour — sending a notification has no return value and
changes no state in our system. The only way to see it is to ask the notifier.

The budget warning test in the walkthrough does **both**: it verifies the interaction (one
notification) and the state (the timestamp was stored).

### 4. Proving that something did NOT happen

R9 says the owner is warned **once**. A test that only checks "the warning is sent at 80 %" would pass
even if the code sent a warning on *every* new entry after 80 %. The owner would get twenty emails.

So half of the important tests check that something did **not** happen:

| Scenario | Assert |
|---|---|
| Value is 79 % | `notify()` never called |
| Warning already sent earlier | `notify()` never called, timestamp unchanged |
| Hourly project, far above any "cap" | `notify()` never called |
| Another request claimed the warning first (row already set, loaded copy still `null`) | `notify()` never called, the other request's timestamp unchanged |
| `notify()` throws (mail server down) | the claim is released: timestamp back to `null`, so a later entry tries again |
| Time entry rejected (archived project) | event not published, nothing saved |

A mock makes this easy: `->never()` (Mockery), `verify(notifier, never())` or `verifyNoInteractions()`
(Mockito), `Event::assertNotDispatched()` (Laravel). The rubric for ÕV6 asks for exactly this: tests
that assert both that something happened **and** that it did not.

### 5. Controlling time

"Now" is an input to your code. Treat it like one.

| Track | Production code asks | Test sets |
|---|---|---|
| Laravel | `now()` (Carbon) | `$this->travelTo(Carbon::parse('2026-10-05 14:30'))`, `$this->freezeTime()`, or `Carbon::setTestNow(...)` |
| Spring | `LocalDateTime.now(clock)` with an injected `java.time.Clock` bean | `Clock.fixed(Instant.parse("2026-10-05T11:30:00Z"), ZoneId.of("Europe/Tallinn"))` |

The two approaches differ in an interesting way:

- **Laravel** changes the time **globally** for the test: every `now()` in the whole application returns
  the travelled-to time until the test ends. Convenient, but invisible — nothing in the class says it
  depends on time.
- **Spring** passes the clock as a **dependency**. The class says "I need a `Clock`" in its constructor,
  and the test hands it a fixed one. More explicit, and it works in plain unit tests with no framework.

`Clock.fixed(...)` is a stub (it returns a prepared answer). Laravel's `travelTo()` is a fake of the
time source. Both make tests deterministic: a test about "one minute in the future" gives the same
result at 03:00 on a Sunday as at 14:00 on a Tuesday, and in CI in another time zone.

Pick boundary times on purpose, like any boundary value: an entry ending **exactly now** (allowed),
**one minute ago** (allowed), **one minute from now** (rejected).

### 6. Framework fakes vs mocking libraries

There are two ways to get a double.

**Framework fakes.** The framework ships ready-made fakes for its own services:

| Laravel | Replaces | You then assert |
|---|---|---|
| `Mail::fake()` | The mailer | `Mail::assertSent(NotificationMail::class, fn ($m) => ...)`, `Mail::assertNothingSent()` |
| `Event::fake([TimeEntryLogged::class])` | The event dispatcher (for these events) | `Event::assertDispatched(...)`, `Event::assertNotDispatched(...)` |
| `Notification::fake()` | Laravel's own notification system | `Notification::assertSentTo(...)` (we do not use Laravel Notifications, but you will meet it at work) |
| `$this->travelTo()` | The clock | — |

Spring has fewer of these; you mostly use **Mockito** with `@Mock` in plain tests or `@MockitoBean` in
slice tests, plus `Clock.fixed()`.

**Mocking libraries** — **Mockery** (PHP, built into Laravel's tests as `$this->mock()`) and **Mockito**
(Java) — create a double for **any** interface or class at runtime. You describe the behaviour:

```php
// Mockery (Laravel): expectation BEFORE the act
$this->mock(Notifier::class, function (MockInterface $mock): void {
    $mock->shouldReceive('notify')->once();
});
```

```java
// Mockito (Spring): stub before, verify AFTER the act
when(projects.findById(7L)).thenReturn(Optional.of(project));
monitor.check(7L);
verify(notifier).notify(anyString(), anyString());
```

Notice the order: Mockery's `shouldReceive()->once()` is set **before** the act (a mock in Meszaros'
sense). Mockito's `verify` is written **after** the act (spy style), which keeps Arrange–Act–Assert in
order. Mockery can do the spy style too (`$this->spy(...)` and `shouldHaveReceived()`).

Use the framework fake when there is one — it knows the framework's details. Use a mocking library for
**your own** interfaces and classes.

### 7. Argument matchers and captors

Often you care *that* a method was called and *roughly* with what, not the exact full string.

- **Matchers** describe acceptable arguments: `Mockery::type('string')`, `Mockery::on(fn ($s) => ...)`;
  Mockito `anyString()`, `eq("x")`, `argThat(s -> s.contains("Website"))`.
- **Captors** grab the real argument so you can assert on it with normal assertions afterwards:
  Mockito `ArgumentCaptor<String>`; in Laravel, a closure in `Event::assertDispatched(..., fn ($e) =>
  ...)` or `Mail::assertSent(..., fn ($mail) => ...)` plays the same role.

```java
ArgumentCaptor<String> subject = ArgumentCaptor.forClass(String.class);
verify(notifier).notify(subject.capture(), anyString());
assertThat(subject.getValue()).contains("Website redesign");
```

How exact should you be? Check what matters to the **user**: the subject names the project. Do not
assert the full message text word by word — then every wording improvement breaks the test.

**Mockito rule:** if one argument uses a matcher, **all** arguments must use matchers.
`verify(notifier).notify("Budget warning", anyString())` fails at runtime; write
`notify(eq("Budget warning"), anyString())`.

### 8. "Don't mock what you don't own"

Imagine testing `BudgetMonitor` by mocking Laravel's `Mail` facade or Spring's `JavaMailSender`
directly. The test would encode assumptions about a library you did not write: which method is called,
in which order, with which objects. When the library changes (or your assumptions were slightly wrong),
the mock still passes and production breaks.

The rule: **mock only types you own.** For third-party code, write a thin adapter with your own
interface, mock *that* interface in the tests of your domain code, and test the adapter itself against
the real library (or the framework's fake) in a few focused tests.

That is exactly the design from lesson 09:

```
BudgetMonitor ──► Notifier (ours)  ◄── mocked in BudgetMonitor tests
                     ▲
                 MailNotifier ──► Mail / JavaMailSender (theirs)
                     ▲
       tested once with Mail::fake() (Laravel) or a mail integration test
```

The `Notifier` interface has one method with two strings. Mocking it is trivial and matches exactly
what `BudgetMonitor` needs. The adapter is small enough to test separately.

### 9. Over-mocking: when mocks hurt

Mocks are powerful, and that makes them easy to overuse. Warning signs:

| Smell | Example | Why it hurts |
|---|---|---|
| **Testing the implementation** | Verifying that `BudgetMonitor` called `findById` exactly once and then `ProjectService.billableMinutes` | A harmless refactor (caching, a different query) breaks the test although behaviour is the same |
| **Brittle exact arguments** | Asserting the complete email text | Every wording change breaks the test |
| **Mocking value objects** | `Mockery::mock(Money::class)`, `mock(Money.class)` | `Money` is pure and fast; a mock removes the real behaviour you depend on and adds setup |
| **Mocking entities / models** | A mocked `Project` with ten stubbed getters | Long setup, and it hides that the entity is just data — build a real one |
| **Mock returns mock returns mock** | `when(a.getB()).thenReturn(b); when(b.getC())...` | The test mirrors the internal structure; the code under test probably reaches too far |
| **Five or more mocks** | A service test with repositories, mailer, logger, clock, config all mocked | The class under test has too many responsibilities — a design problem, not a test problem |

A good mock-based test mocks **only the boundaries** — the notifier, the event publisher, the clock,
maybe a repository — and uses real objects for everything in the middle.

### 10. When a hand-written fake beats a mock

A **fake** is a small working implementation written for tests:

```php
final class InMemoryNotifier implements Notifier
{
    /** @var list<array{subject: string, message: string}> */
    public array $sent = [];

    public function notify(string $subject, string $message): void
    {
        $this->sent[] = ['subject' => $subject, 'message' => $message];
    }
}
```

Compare two tests of "warned only once when `check()` runs twice":

| | Mock | Fake |
|---|---|---|
| Setup | `shouldReceive('notify')->once()` (or `verify(..., times(1))`) | `new InMemoryNotifier` |
| Assert | Implicit in the expectation | `assertCount(1, $notifier->sent)` — plain state check |
| Checking the subject | A matcher inside the expectation | Normal assertion on `$notifier->sent[0]['subject']` |
| Failure message | "Method notify() should be called exactly 1 times but called 2 times" | "Failed asserting that actual size 2 matches expected size 1" — plus you can print the messages |
| Reuse | Written again in every test | One class, used by many tests |

Fakes shine when many tests need the same double, when you want to look at *everything* that was sent,
or when the collaborator has state (an in-memory repository that remembers what you saved). Mocks shine
for one-off checks like "never called" or "called with exactly this argument". Neither is always better
— and being able to explain the choice is part of ÕV6 at the *exceeds* level.

### 11. Where doubles meet dependency injection

How does the double get into the code under test?

| Situation | Laravel | Spring |
|---|---|---|
| Plain unit test | `new BudgetMonitor($fakeNotifier, ...)` | `new BudgetMonitor(projects, projectService, new HourlyBilling(), notifier, clock)` in `@BeforeEach` |
| Framework builds the object | `$this->mock(Notifier::class)` / `$this->app->instance(Notifier::class, $fake)` replaces the binding in the container; `app(BudgetMonitor::class)` then receives the double | `@MockitoBean ClientService` replaces the bean in the Spring test context (`@WebMvcTest`) |

Either way, the double arrives through the **constructor**. If a class calls `new` on its collaborators
or pulls them from a global, you cannot swap them — the test is telling you to fix the design.

---

## Laravel vs Spring Boot

| Idea | Laravel | Spring Boot |
|---|---|---|
| Mocking library | Mockery (`$this->mock()`, `$this->spy()`) | Mockito (`@Mock`, `@ExtendWith(MockitoExtension.class)`) |
| Replace a container binding / bean | `$this->mock(X::class)`, `$this->app->instance(X::class, $obj)` | `@MockitoBean` (Boot 4; replaces `@MockBean`) |
| Stub a return value | `->shouldReceive('m')->andReturn($x)` | `when(mock.m()).thenReturn(x)` |
| Expect a call | `->shouldReceive('m')->once()` (before act) | `verify(mock).m()` (after act) |
| Expect no call | `->shouldReceive('m')->never()`, `shouldNotReceive('m')` | `verify(mock, never()).m(...)`, `verifyNoInteractions(mock)` |
| Capture an argument | `Mockery::on(fn ($arg) => ...)` or assertion closure | `ArgumentCaptor` / `@Captor` |
| Control time | `travelTo()`, `freezeTime()`, `Carbon::setTestNow()` | Inject `Clock`; `Clock.fixed(...)` in tests |
| Mail | `Mail::fake()` + `assertSent` | Mock your `Notifier`; test `MailNotifier` against a real SMTP server (optional integration test, not built in this course) |
| Events | `Event::fake([...])` + `assertDispatched` | Mock `ApplicationEventPublisher` + `verify(...).publishEvent(...)` |
| Controller test | Feature test: `$this->post(...)`, `assertRedirect`, `assertSessionHasErrors` | `@WebMvcTest` + `MockMvcTester` + `@MockitoBean` |

---

## Common mistakes

| Mistake | Why it hurts | What to do instead |
|---|---|---|
| Only testing that the warning **is** sent | A bug that sends it on every entry passes | Also test "not at 79 %", "not twice", "not for hourly" |
| Mocking `Money`, `Project`, `TimeRounding` | Setup grows, real behaviour disappears | Use real value objects and real entities built in memory |
| Mocking the `Mail` facade / `JavaMailSender` in domain tests | Assumptions about code you don't own | Mock your `Notifier`; test the adapter separately |
| Asserting the full email text | Every wording change breaks the test | Assert what matters (the subject names the project) |
| `Event::fake()` with no class list | Also fakes Eloquent model events, which can break models that rely on them | `Event::fake([TimeEntryLogged::class])` |
| Creating the object under test **before** `$this->mock(...)` (Laravel) | It already holds the real notifier; the mock is never used | Register the mock first, then `app(BudgetMonitor::class)` |
| Stubbing things the test never uses (Mockito) | `UnnecessaryStubbingException`; noise | Stub only what the scenario needs |
| `@MockBean` from an old tutorial | Removed in Spring Boot 4 | `@MockitoBean` from `org.springframework.test.context.bean.override.mockito` |
| Calling `now()` / `LocalDateTime.now()` without control in a tested class | Flaky tests around midnight and weekends | Laravel: `travelTo()`; Spring: inject `Clock` |
| Verifying every call (`verifyNoMoreInteractions` everywhere) | Brittle: any extra harmless call breaks tests | Verify the calls that are the behaviour |
| A test with six mocks | The test mirrors the implementation | Split the class, or use fakes for state-heavy collaborators |

---

## Independent work (~2 h)

Do these at home after the walkthrough, in your own project (~2 h). Tasks build on each other across lessons, so finish at least the **Basic** ones. Solutions are at the bottom of the [Laravel](laravel.md#independent-work--solutions) and [Spring Boot](spring-boot.md#independent-work--solutions) walkthroughs — try first, compare after.

### Task 1 — Tests for every `TimeEntryService` rule (Basic)

Write tests for `TimeEntryService::log()` / `TimeEntryService.log()` that cover every rule it owns.

Acceptance criteria:

- [ ] A valid entry is saved **and** the `TimeEntryLogged` event (Spring: `TimeEntryLoggedEvent`) is published once, for the right
      project.
- [ ] An archived project is rejected with `ProjectArchivedException`: nothing is saved and **no**
      event is published.
- [ ] A project id that does not exist is rejected (Laravel `ModelNotFoundException`, Spring
      `NotFoundException` — as in lesson 07; both become a 404); no event.
- [ ] Laravel: feature tests with `RefreshDatabase` and `Event::fake([TimeEntryLogged::class])`.
      Spring: a Mockito unit test with mocked repositories and a mocked `ApplicationEventPublisher`.
- [ ] Every rejection test asserts **both** the exception and the absence of side effects.

### Task 2 — The future-entry rule with a controlled clock (Basic)

Acceptance criteria:

- [ ] The clock is fixed (Laravel `travelTo`, Spring `Clock.fixed`) at a known moment.
- [ ] Three boundary cases: the entry ends one minute before "now" (saved), exactly at "now" (saved),
      one minute after "now" (rejected with `TimeEntryInFutureException`, no event).
- [ ] The tests pass on any day, at any time, in CI.

### Task 3 — A controller test for creating a client (Intermediate)

Acceptance criteria:

- [ ] Valid input → redirect to the new client's page with the flash message.
- [ ] Invalid input (blank name, bad e-mail) → the form is shown again with errors on those fields; the
      service/database is not touched.
- [ ] Duplicate e-mail → the form is shown again with an error on `email`; no second client exists.
- [ ] Laravel: feature test (`$this->post(...)`, `assertRedirect`, `assertSessionHasErrors`).
      Spring: `@WebMvcTest(ClientController.class)` with `MockMvcTester` and `@MockitoBean ClientService`.

### Task 4 — Refactor an over-mocked test into a fake-based one (Advanced)

Acceptance criteria:

- [ ] Pick one of your tests with the most mocking (or write the deliberately over-mocked example given
      in the walkthrough) and rewrite it with a hand-written fake (`InMemoryNotifier` or an in-memory
      repository).
- [ ] In `docs/testing.md`, add a section "Mock or fake?" (5–10 sentences, English): what the old test
      checked, why it was brittle, what the new one checks, and one case where you **kept** a mock and
      why.

---

## Self-check

1. What is the difference between a stub and a mock? Give a Billable example of each.
2. Why does the budget warning need tests that check something did **not** happen?
3. What is state verification, and what is interaction verification? When is interaction verification
   the only option?
4. Why do we mock `Notifier` and not the `Mail` facade / `JavaMailSender`?
5. How do you make the rule "an entry cannot end in the future" testable in your track?
6. A test mocks `Money`. What do you tell the author?
7. When would you write a fake instead of using a mocking library?
8. In the defence you are asked: "This test uses a mock. What would break if you used the real class?"
   Answer it for the `BudgetMonitor` test.

<details>
<summary>Answers</summary>

1. A stub returns prepared answers and is not checked — `findById(7)` returning a test project, or a
   fixed clock. A mock has expectations about how it is called, and the test fails if they are not met —
   a notifier that must receive `notify()` exactly once.
2. Because R9 says "once". Code that warns on every entry after 80 % satisfies "the warning is sent at
   80 %" but spams the owner. Only a test that expects **no** second call catches it. The same for 79 %
   and hourly projects.
3. State verification checks results and object state after the act (the returned amount, the stored
   timestamp). Interaction verification checks calls to collaborators (notify called once). When the
   behaviour is a side effect that leaves no state in our system — sending a message — interaction
   verification is the only way to see it.
4. We own `Notifier`; we do not own the mail library. Mocking the library encodes guesses about its API
   and breaks (or silently passes) when it changes. Our interface is tiny and stable, and `MailNotifier`
   is tested separately against the framework's mail fake.
5. Laravel: the service uses `now()`, and the test calls `$this->travelTo(...)` to fix "now". Spring: the
   service uses `LocalDateTime.now(clock)` with an injected `Clock`, and the test passes
   `Clock.fixed(...)`.
6. `Money` is a pure value object. Use the real one. A mock needs setup, removes real behaviour (the test
   no longer checks the arithmetic it relies on) and breaks when `Money` gains a method.
7. When several tests need the same double, when the collaborator has state (read-after-write), or when
   you want to inspect everything that happened with normal assertions. Mocks are better for one-off
   interaction checks such as "never called".
8. The real `MailNotifier` would send a real email to Mailpit (or fail if Mailpit is not running), the
   test would be slower, and — most important — the test could not easily check *how many* warnings were
   sent. With the mock we can assert "exactly once", and "never" for 79 %, which is the rule R9.

</details>

---

## Checklist

- [ ] Budget warning tests: sent at 80 %, not at 79 %, not twice, not for hourly — notifier replaced.
- [ ] At least one test asserts an interaction (called once) and at least one asserts its absence
      (never called / not dispatched).
- [ ] The clock is controlled wherever the code under test uses "now".
- [ ] `TimeEntryLogged` publication tested, including "not published when rejected".
- [ ] One hand-written fake (`InMemoryNotifier`) used in a test.
- [ ] Laravel: `Mail::fake()` test for `MailNotifier`. Spring: one `@WebMvcTest` with `@MockitoBean`.
- [ ] No value objects or entities are mocked.
- [ ] You can explain dummy, stub, spy, mock and fake with examples from your own tests. (ÕV6 —
      *kasutab testides mock-klasse*)
- [ ] All tests green locally and in CI.

---

## Further reading

- [Laravel — Mocking](https://laravel.com/docs/mocking)
- [Laravel — Mail: testing mailables](https://laravel.com/docs/mail#testing-mailables)
- [Laravel — Events: testing](https://laravel.com/docs/events#testing)
- [Mockery documentation](https://docs.mockery.io/)
- [Mockito documentation (Javadoc)](https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/Mockito.html)
- [Spring Framework — MockMvc and MockMvcTester](https://docs.spring.io/spring-framework/reference/testing/mockmvc.html)
- [Spring Boot — Testing](https://docs.spring.io/spring-boot/reference/testing/index.html)
- [Martin Fowler — Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html)
- [Martin Fowler — Test Double](https://martinfowler.com/bliki/TestDouble.html)
