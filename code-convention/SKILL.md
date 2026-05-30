---
name: code-convention
description: "Apply battle-tested code conventions when writing or modifying code in Laravel/PHP projects (with React/TypeScript frontend support). Use this skill whenever the user is writing a new feature, refactoring existing code, creating a controller/action/migration/model/test, or asks 'how should I structure this', 'what's the convention for X', 'is this the right pattern'. Also use proactively before producing non-trivial code so the output follows conventions the first time instead of needing rewrites."
---

# Code Convention

A reference of conventions for writing maintainable Laravel/PHP code (with React/TypeScript frontend). Apply these when authoring or modifying code — not as a post-hoc review checklist, but as the default shape of the code you produce.

## How to use this skill

1. Skim the relevant section(s) for the code you're about to write (Architecture for new features, Database for migrations, Testing for tests, etc.).
2. Match your output to the patterns shown. When a project-specific convention conflicts with this skill (check `CLAUDE.md`, existing code in the area you're modifying), the project wins — these are sensible defaults, not absolute laws.
3. If you find yourself violating a rule, pause and justify *why*. Most violations are accidental; the few intentional ones usually have a good reason worth stating in a comment or PR description.

The "why" matters more than the rule. Each section explains the reasoning so you can apply judgment to edge cases rather than mechanically following text.

---

## Architecture

### Keep business logic out of controllers

Controllers handle HTTP — request parsing, response shaping, status codes. Business logic belongs in **Actions** (mutations) and **Queries** (reads). The reason: anything more than a trivial controller method is usually needed by other entry points too (API endpoints, mobile apps, console commands, queued jobs). Logic locked inside a controller method has to be duplicated or extracted later.

```php
// Query — for reads
final class CourseReviewSummaryQuery
{
    public function execute(Course $course): array { /* ... */ }
}

// Action — for mutations
final class CreateEnrollmentAction
{
    public function execute(Course $course, Student $student): Enrollment { /* ... */ }
}
```

A controller method with 1–2 simple queries plus an Action call is fine. Extract to an Action when you have: multiple mutations, orchestration across services, or logic that will plausibly be reused.

### Actions and Queries hold execution, not guards

An Action/Query is the **execution step**. By the time data reaches it, that data is already trusted: shape-validated, type-cast, authorized, and business-rule-checked. The Action's job is to *do the thing*, not to argue about whether the thing should be done.

This means **no guards inside Actions or Queries** — no auth checks, no "does this user own this record" checks, no "is this status transition allowed" checks, no defensive `throw` statements for conditions the caller could have prevented. All of that belongs in the layer that calls the Action: the controller, the parent Action, the command handler, the job — whoever owns the decision to invoke it.

```php
// AVOID — guards mixed into the Action
final class PublishCourseAction
{
    public function execute(Course $course, User $actor): void
    {
        if ($actor->cannot('publish', $course)) {
            throw new AuthorizationException();
        }
        if ($course->status !== CourseStatus::Draft) {
            throw new InvalidStateException('Only draft courses can be published.');
        }
        if ($course->chapters()->count() === 0) {
            throw new ValidationException('Course needs at least one chapter.');
        }

        $course->status = CourseStatus::Published;
        $course->published_at = now();
        $course->save();
    }
}

// PREFERRED — Action is purely the execution
final class PublishCourseAction
{
    public function execute(Course $course): void
    {
        $course->status = CourseStatus::Published;
        $course->published_at = now();
        $course->save();
    }
}

// Guards live where the decision is made — the controller, policy, form request, or parent action
public function publish(Course $course, PublishCourseAction $action): RedirectResponse
{
    $this->authorize('publish', $course);                          // authorization
    Gate::denyIf($course->status !== CourseStatus::Draft);         // state guard
    Gate::denyIf($course->chapters()->doesntExist(), 'No chapters'); // business rule

    $action->execute($course);

    return redirect()->route('courses.show', $course);
}
```

Why this matters:

- **One responsibility per layer.** Mixing validation with execution makes Actions hard to test (every test has to satisfy the guards before reaching the behavior under test) and hard to reuse (a queued job, a console command, an admin override may have *different* guard rules — but the same execution).
- **Guards move toward the boundary.** Authorization belongs in Policies/Form Requests; state checks belong with the business decision; shape validation belongs in the Request/DTO. Pushing them into the Action duplicates them or hides them.
- **Exceptions stop being a control-flow tool.** When Actions throw guard exceptions, callers start catching them to branch on — and now the "happy path" and "rejected path" both flow through `try/catch`. Decisions made up front in plain `if` statements are easier to read and easier to test.

If you find yourself wanting to put a guard inside an Action, the question to ask is: *who knew this could fail, and why didn't they check before calling me?* That's where the guard belongs.

### Create models with `new Model()` + property assignment, not `Model::create([...])`

```php
// PREFERRED
$chapter = new Chapter();
$chapter->course_id = $chapterData->courseId;
$chapter->title = $chapterData->title;
$chapter->save();

// PREFERRED — relationships via associate()
$progress = new ChapterItemProgress();
$progress->chapterItem()->associate($chapterItem);
$progress->student()->associate($student);
$progress->save();

// AVOID
Chapter::create(['course_id' => $id, 'title' => $title]);
```

Property assignment is type-safe, IDE-navigable, and surfaces type mismatches at write-time rather than at runtime. Mass assignment via arrays bypasses these guarantees and makes it easy to silently drop fields.

### Pass objects, not primitives

```php
// AVOID
public function execute(int $courseId): void

// PREFERRED
public function execute(Course $course): void
```

Passing the model gives you the full object with its relationships and methods. Passing an ID forces the receiver to refetch — extra queries, extra "what if it's been deleted" cases, and the type system can't help you confuse a `course_id` with a `student_id`.

Only fall back to primitives when you genuinely don't have the object (e.g., a queued job that must serialize, where the model might not exist by the time it runs).

### Never use `Auth::user()` (or any auth facade) inside models

Models should be pure data + behavior. Reaching into the auth facade from a model couples it to the HTTP request lifecycle — the same model won't work in console commands, queued jobs, or tests without ceremony. Pass the user or actor in as a parameter.

### Use model scopes instead of raw `where()` when one exists

```php
// Given: scopeForStudent(Builder $query, Student|int $student)

// PREFERRED
$enrollments = Enrollment::query()->forStudent($student)->get();

// AVOID
$enrollments = Enrollment::query()->where('student_id', $student->id)->get();
```

Scopes carry domain meaning and centralize filter logic. When the filter rule changes (soft-deletes, tenant scoping, status filters), the scope updates everywhere.

### Don't conflate concerns in a single field

If a field is doing two jobs — say `status` that also encodes visibility — split it. The single field looks tidy until you need to query "all published-but-archived items" and find you can't.

### Prefer generic solutions over platform-specific code

When you can configure a behavior instead of branching on platform/vendor/provider, do that. Only specialize when the behaviors are genuinely different and unlikely to converge.

### Reuse existing endpoints before creating new ones

If an endpoint serves the data you need, extend or reuse it. Parallel endpoints that return overlapping data drift apart over time.

### [Strict] Resolve dependencies by injection, never with the `app()` helper

When you need an instance of a class — an Action, a Query, a service, a repository — **inject it**. Type-hint it as a constructor parameter (for classes the container builds) or a method parameter (controllers, jobs, command `handle()` methods), and let the service container hand it to you.

```php
// PREFERRED — constructor injection
final class CheckoutController extends Controller
{
    public function __construct(
        private readonly ProcessPaymentAction $processPayment,
    ) {}

    public function store(StoreCheckoutRequest $request): RedirectResponse
    {
        $this->processPayment->execute(CheckoutData::from($request->validated()));
        return redirect()->route('orders.index');
    }
}

// PREFERRED — method injection (controllers, jobs, commands)
public function store(StoreCheckoutRequest $request, ProcessPaymentAction $action): RedirectResponse
{
    $action->execute(CheckoutData::from($request->validated()));
    return redirect()->route('orders.index');
}

// AVOID — the app() helper
$action = app(ProcessPaymentAction::class);
$action->execute($data);

// AVOID — new-ing it up bypasses the container entirely
$action = new ProcessPaymentAction(new PaymentGateway(config('services.stripe.secret')));
```

Why injection over `app()`:

- **The dependency is visible in the signature.** Anyone reading the constructor sees exactly what the class needs. `app()` calls are scattered through method bodies, hiding the real dependency graph and making the class lie about what it requires.
- **It's testable.** Injected dependencies can be swapped for fakes/mocks in tests without touching the container. `app()` reaches into global state, so a test has to bind into the container to control it — more setup, more coupling.
- **It fails loudly and early.** If the container can't build an injected dependency, you find out at resolution time with a clear stack trace, not deep inside a method when `app()` finally runs.

If a situation genuinely makes injection impossible — resolving conditionally at runtime, inside a closure the container doesn't manage, breaking a circular dependency — fall back to `App::make(...)` (the explicit facade, with a `use Illuminate\Support\Facades\App;` import) rather than the `app()` helper. Treat this as the rare exception, not a convenience: if you're reaching for it, first ask whether the class could have been injected instead. Avoid `new Class()` for anything with dependencies — it bypasses the container, so bindings, singletons, and decorators don't apply.

---

## DTOs and Request Validation

### Use DTOs (Spatie Laravel Data), not arrays, for Action inputs
### Validate input with Laravel Form Request classes — not DTOs, not inline rules

Use the `Data` suffix (e.g., `StudentProfileData extends Data`). Passing arrays into Actions defeats the whole point: you lose types, lose IDE support, lose validation at the boundary, and gain nothing.
Validation at the HTTP boundary belongs in a dedicated **Form Request** class (`StoreCommentRequest extends FormRequest`). Don't use:

### Always use validated data, never raw input
- **Spatie Laravel Data / other DTO libraries for validation.** DTOs are transport objects. Coupling validation rules into the same class that travels around the application mixes two concerns — the validation runs only at the boundary, but the rules live on a class that gets passed into Actions, jobs, and tests where they're irrelevant noise.
- **Inline `$request->validate([...])` in the controller.** Validation logic in the controller body bloats the method, hides the contract of the endpoint, and can't be reused or extended (e.g., adding authorization, custom messages, `prepareForValidation`). The controller should describe *what to do once data is valid*, not *what valid data looks like*.

```php
// PREFERRED — Form Request
// PREFERRED — Form Request owns the rules, controller is clean
public function store(StoreCommentRequest $request, StoreCommentAction $action): RedirectResponse
{
    $validatedData = $request->validated();
    $action->execute($validatedData);
    $action->execute(CommentData::from($request->validated()));
    return redirect()->back();
}

// PREFERRED — inline validation
// AVOID — inline validation in the controller
public function store(Request $request, StoreCommentAction $action): RedirectResponse
{
    $validatedData = $request->validate([
        'text' => ['required', 'string', 'max:500'],
    ]);
    $action->execute($validatedData);
    return redirect()->back();
}

// AVOID
// AVOID — raw input, no validation contract
$action->execute($request->input('text'));
```

`$request->input()` returns whatever the client sent, including fields you didn't ask for and types you didn't expect. `validated()` returns only the allow-listed, cast values.
Why Form Requests:

- **One place to find the contract.** A reader looking up "what does this endpoint accept?" opens one file with `rules()`, `authorize()`, custom messages, and `prepareForValidation()` all together.
- **Authorization belongs with validation.** `authorize()` runs before `rules()`. Putting both on the Form Request keeps the gate next to the door.
- **Composable.** Form Requests can extend a base class, share rules, override messages per locale, and be unit-tested without booting the controller.
- **`$request->input()` returns whatever the client sent**, including fields you didn't ask for and types you didn't expect. `$request->validated()` returns only the allow-listed, cast values.

### Don't force type conversions inside the Request class
The one acceptable exception is a *trivial* endpoint with no inputs beyond a route parameter (e.g., a `destroy` action that takes only the bound model). No Form Request is needed there because there's nothing to validate.

Request classes describe *what's valid*. If you need to transform a string into an enum, parse a date, or hydrate a DTO, that belongs in the Action — where the conversion is testable and reusable.
### DTOs are for transport into Actions, not for validation

Once data is validated by the Form Request, hydrate a DTO (e.g., Spatie Laravel Data with the `Data` suffix: `CommentData extends Data`) from the validated array and pass *that* to the Action. The DTO carries typed, structured data through the application. It does **not** carry validation rules — those stayed at the boundary, in the Form Request, where they belong.

Passing raw arrays into Actions defeats the point: you lose types, lose IDE support, and force every Action to re-derive what shape its input has.

### Don't force type conversions inside the Form Request

Form Requests describe *what's valid*. If you need to transform a string into an enum, parse a date, or hydrate a DTO from the validated array, that belongs in the Action (or in the DTO's own constructor / `from()` method) — where the conversion is testable and reusable across entry points that don't go through the same Form Request.

### Send actual booleans from the frontend

Send `true`/`false`, not `"true"`/`"1"`/`"0"`. The string-vs-boolean confusion at the boundary is a forever source of bugs.

---

## API Responses

### Use API Resources for reusable/complex response structures

```php
// PREFERRED — Resource for reusable/complex data
public function index(): AnalyticsIntegrationsResource
{
    return new AnalyticsIntegrationsResource($data);
}

// OK — simple message responses don't need a Resource
return response()->json(['message' => 'Device logged out successfully']);
return response()->json(['message' => $errorMessage], 403);
```

Resources are for structured data returned from multiple places or containing nested data. A one-line `['message' => '...']` doesn't need ceremony.

### HTTP methods, not verbs in URLs

Routes follow REST. The HTTP method carries the action; the URL identifies the resource.

```php
// AVOID
Route::post('payment/{id}/delete', 'delete');

// PREFERRED
Route::delete('payment/{id}', 'destroy');
```

Use `PUT` for full updates and `PATCH` for partial. Keep response keys consistent across endpoints (`data`, `meta`, `errors`).

---

## Enums

Don't use `->value` in PHP code. The whole point of an enum is type safety; `->value` throws it away to get a string, which then has to be re-validated everywhere it lands. Use the enum directly in PHP and let Eloquent's enum casts handle persistence.

`->value` is fine in Blade and JavaScript — those layers genuinely need the primitive.

---

## File Organization

- **No `Traits/` or `Interface/` folders.** These classify by *language construct* rather than by *concern*. A trait belongs next to the code that uses it; an interface belongs with its primary implementation or in the domain folder it describes. We don't have a `Class/` folder — same reasoning.
- **Lowercase root folders** (`app/actions/`, `app/queries/`).
- **Place things where they'll be used**, not where the type system would put them.
- **Snake_case for Blade files** (`google_meet.blade.php`).

---

## Naming

- Method names should match what they return: `has*` → bool, `is*` → bool, `get*` → object/value, `find*` → object or null.
- Method names start with a verb.
- Use real English words (`unread`, not `un_read`).
- Avoid generic class names. `Onboarding` is too vague — `OnboardingPreference` says what it actually is.
- Avoid confusingly similar names in the same area: `CreateAction` next to `CreateNewAction` is a trap waiting to happen.
- Don't put the file-type suffix in the class name: prefer `Export` over `ExportXlsx` (the format is an implementation detail).
- Artisan/console commands: use a project-wide prefix (e.g., `myapp:command-name`). Reserve a different prefix (e.g., `disposable:`) for one-time scripts so they're easy to find and prune later.

---

## Events

Use the framework's event auto-discovery rather than registering listeners manually. Manual registration of an auto-discovered listener causes it to fire twice — a class of bug that's painful to debug after the fact.

Only register manually when the listener extends or implements a base class that auto-discovery can't see. When you do, **leave a comment** explaining why so the next person doesn't "clean it up".

---

## Testing

```php
use PHPUnit\Framework\Attributes\Test;

final class CreateCourseActionTest extends TenantTestCase
{
    #[Test]
    public function creates_course_with_valid_data(): void
    {
        // Arrange
        $instructor = User::factory()->create();
        $action = $this->app->make(CreateCourseAction::class);

        // Act
        $course = $action->execute(/* ... */);

        // Assert
        $this->assertSame('Expected Title', $course->title);
    }
}
```

- **`#[Test]` attribute + snake_case method names** — reads as natural-language behavior descriptions.
- **`$this->app->make(...)`** for the system under test, not `new Class()`. The container injects dependencies and surfaces wiring issues your tests should catch.
- **Factories** for test data. Inline `Model::create([...])` in tests duplicates schema knowledge across hundreds of files.
- **Duplication in arrange steps is OK.** Two tests with similar setup are easier to read and modify independently than one test backed by a shared helper that takes seven arguments. Extract only when the duplication actually hurts.
- **No useless tests.** A test that would pass on a broken implementation is worse than no test — it provides false confidence and adds maintenance load.

### Every test stands alone — no shared arrange helpers

Treat each test as a self-contained piece of code. The **arrange** step (creating users, tenants, courses, fixtures, whatever the test needs) is written *in the test itself*, even when several tests in the file need almost the same setup.

That means **no** reusable test helpers for arrange logic:

- No `setUpCourseWithChapters()` private methods on the test class.
- No `WithBillingScenario` / `CreatesEnrolledStudents` traits mixed into many test classes.
- No shared `TestDataBuilder` / `Scenario` objects that hide what's actually being created.

Factories (e.g., `Course::factory()->create([...])`) are fine — they're the per-model construction primitive. What we avoid is the *next layer up*: helpers that compose multiple factories into business scenarios and get shared across tests.

```php
// AVOID — shared trait hides what each test is actually testing
trait CreatesEnrolledStudent
{
    protected function enrolledStudent(Course $course = null): Student
    {
        $course ??= Course::factory()->published()->create();
        $student = Student::factory()->create();
        Enrollment::factory()->for($student)->for($course)->create();
        return $student;
    }
}

#[Test]
public function completes_chapter(): void
{
    $student = $this->enrolledStudent();
    // ... what course? what state? reader has to jump to the trait
}

// PREFERRED — every relevant fact lives in the test
#[Test]
public function completes_chapter(): void
{
    // Arrange
    $course = Course::factory()->published()->create();
    $chapter = Chapter::factory()->for($course)->create();
    $student = Student::factory()->create();
    Enrollment::factory()->for($student)->for($course)->create();

    // Act
    $this->app->make(CompleteChapterAction::class)->execute($student, $chapter);

    // Assert
    $this->assertDatabaseHas('course__chapter_progress', [
        'student_id' => $student->id,
        'chapter_id' => $chapter->id,
        'completed_at' => now(),
    ]);
}
```

Why duplication in tests is the *right* tradeoff:

- **A test is documentation of one behavior.** When the arrange step is inline, the reader sees every precondition that matters. With a shared helper, the meaningful preconditions are buried under abstraction and the test reads as "do the magic setup, then assert".
- **Shared test helpers couple unrelated tests.** Today's `enrolledStudent()` returns a published course. Tomorrow someone needs a draft course. They add a flag. Six months later the helper takes nine optional arguments, every change to it risks breaking distant tests, and nobody can read it.
- **The cost of duplication in tests is low.** Tests rarely "refactor" — when a test fails after a code change, you usually want to update that one test, not propagate a change through a shared scenario builder. Duplicated arrange code makes each test independent to update.
- **Tests are read far more than they're written.** Optimize for the reader landing on a single failing test and understanding it from top to bottom, not for the writer saving a few lines.

If the same arrange block genuinely repeats across many tests in a single file, a small private method *on that test class* (not a trait, not a shared base) is acceptable — but the default is to write it out every time. The pain of duplication is the signal that tells you when the underlying production code is missing a useful seam, not a signal to build a test-side abstraction.

---

## Security

- Sanitize user-generated HTML on the way out. In Laravel projects this typically means a cast like `PurifyHtmlOnGet::class` on fields that hold rich text.
- Every controller method (and equivalent API endpoint) needs authorization. The default should be "deny", not "the route name was hard to guess".
- Don't write `$tenant?->...` when tenant is required by the surrounding code. The null-safe operator there hides a real bug and turns an exception into wrong behavior.
- Never commit real secrets to `.env.example`. Use clearly fake placeholders.

---

## Code Quality

- **No debug code in commits**: `dd()`, `dump()`, `var_dump()`, `console.log()`, `Log::debug()` left in for "just this once".
- **No commented-out code.** Git remembers; the file shouldn't.
- **Count in the query, not after fetching:**
  ```php
  // AVOID
  $orders->where(...)->get()->count();

  // PREFERRED
  $orders->where(...)->count();
  ```
- **Don't override core Eloquent methods** like `getKey()` unless you really know what you're doing — too much framework code depends on the default behavior.
- **Wrap external API calls in try/catch.** Third-party services fail; your code shouldn't 500 because their DNS hiccupped.
- **Document when you remove a global scope** (tenant scope, soft-delete scope). Removing it is sometimes correct — in background jobs, admin tooling — but the next reader needs to know it was intentional.
- **Native PHP types over PHPDoc.** `function foo(string $name): User` over a `@param` comment. PHPDoc lies; types are checked.
- **Three copies is wrong.** Two might be coincidence; three is a pattern asking to be extracted.

---

## Comments

Default to writing **no comments**. Well-named identifiers, small functions, and clear control flow are the real documentation — they can't drift out of sync with the code because they *are* the code. A comment is a second source of truth competing with the first, and the first will win every time a future change forgets to update the second.

### When a comment is worth adding

Only add one when the *why* is non-obvious from reading the code:

- **A hidden constraint or invariant** the type system can't express ("this list is sorted by `created_at desc` because the consumer relies on the first match").
- **A workaround for a specific bug or limitation** in an external system, with enough context to delete the workaround later ("Stripe returns 402 here on duplicate idempotency keys — treat as success").
- **Surprising behavior** that would look like a mistake to a reasonable reader ("yes, we intentionally swallow this exception — see [ticket / incident]").
- **A non-obvious performance choice** ("inlined to avoid an N+1; benchmarked at 12× faster on a 10k-row dataset").
- **Required by domain or regulation** ("field stored hashed for GDPR right-to-erasure — see DPIA §4").

If you can remove the comment without confusing a future reader, remove it.

### What not to comment

- **What the code does.** `// increment the counter` next to `$counter++` is noise. If a block is opaque enough to need narration, the fix is usually a clearer name or a smaller function, not a comment.
- **The current task, fix, or callers.** "Added for the X flow", "used by Y page", "fixes ticket #123" — that context belongs in the PR description and the commit message. Files outlive PRs; references rot.
- **Commented-out code.** Git remembers. Leaving dead code in the file makes readers wonder if it's supposed to come back, and tools can't refactor across it.
- **Restating the type signature.** A `@param string $name` over `function greet(string $name)` adds zero information and one more thing to keep in sync.
- **Vague TODOs.** `// TODO: improve this later` with no owner, no condition, and no detail is a wish, not a task. Either describe what specifically needs to change and when (`// TODO: replace with the new API once vendor ships v2 — tracked in #4421`) or delete it.

### Examples

```php
// AVOID — narrates the code
// Loop through users and send them an email
foreach ($users as $user) {
    Mail::to($user)->send(new Welcome());
}

// AVOID — restates the signature
/**
 * @param User $user
 * @return string
 */
public function fullName(User $user): string

// GOOD — captures a non-obvious constraint
// Must run before BillingReset because it relies on the previous cycle's invoices
// still being present in the working table.
public function execute(): void

// GOOD — captures a workaround with context
// Provider returns 200 with body { error: ... } instead of a 4xx on validation
// failures; check the body, not the status. See vendor ticket SUP-8821.
if (isset($response->body->error)) {
    throw new ProviderValidationException($response->body->error);
}
```

### Docblocks for public APIs

Library/package code and SDK surfaces are the exception — a class or method that other teams depend on benefits from a docblock describing intent, parameters that aren't self-evident, and thrown exceptions. Keep them factual; don't pad with redundant tags the type signature already carries.

---

## Blade

- PHP `@php` blocks live at the top of the file, not interleaved.
- Business logic goes in ViewModels or computed properties, not in the template.
- Use components (`<x-button>`) for repeated UI; use partials for repeated chunks of markup.
- Prefer CSS `text-transform: uppercase` over hardcoded uppercase text — keeps translation/localization sane.

---

## Frontend (React / TypeScript)

- **No `setTimeout` to "wait for" API calls.** If you find yourself reaching for it, the real fix is awaiting the right promise, using the right query state, or invalidating the right cache.
- **No `alert()` / `confirm()` / `prompt()`.** Use the project's toast/modal/notification components — those are styled, accessible, and testable.
- **Validate query keys** for your data-fetching library so refetches and invalidation actually hit. A typo in a key is a silent cache miss.
- **Persist form state with fallbacks** — when the user navigates away and back, they shouldn't lose their work to a hydration race.
- **Gate features by who should see them** (paid, trial, admin) rather than rendering and disabling.

### Never use emojis as icons

An emoji (🔍 ✅ ⚙️ 🗑️) is not an icon. Use a real icon from the project's icon library (Lucide, Heroicons, Font Awesome, a local SVG sprite — whatever the project standardized on). Emojis render inconsistently across platforms and fonts (Apple, Windows, Android, and Linux each draw them differently), can't be styled with `currentColor` / size utilities, carry no reliable accessible name, and look amateur next to a real icon set. They're for human-written text (chat, copy, commit messages), not UI chrome.

```jsx
// AVOID — emoji standing in for an icon
<button>🗑️ Delete</button>
<span>✅ Saved</span>

// PREFERRED — icon component from the library
import { Trash2, Check } from 'lucide-react';

<button><Trash2 className="size-4" aria-hidden /> Delete</button>
<span><Check className="size-4 text-green-600" aria-hidden /> Saved</span>
```

The same applies in Blade (`<x-icon name="trash" />` or an `@svg` directive) — reach for the icon component, not a literal emoji character.

If the project has **no** icon library, don't reach for an emoji as a shortcut. Either:
1. **Ask the user to install one** (e.g., `lucide-react`, Blade Icons) and use it — this is the preferred path since the whole app then shares one consistent, styleable set; or
2. **Inline an SVG** for the one icon you need right now, ideally pulled from an open icon set so it matches the rest visually.

Flag the missing library to the user rather than silently working around it — picking the icon system is a project-level decision worth surfacing.

---

## Config

- External services in `config/services.php` (or the framework equivalent), not scattered across feature configs.
- `snake_case` for config keys, consistently — never mix `snake_case` with `kebab-case` in the same file.
- Reference env vars only inside config files. The rest of the app reads `config(...)`, never `env(...)`.

---

## Routes & URLs

- REST principles: no action verbs in route URLs.
- Group related routes (shared prefix, middleware, name prefix).
- Standard naming: `index`, `show`, `store`, `update`, `destroy`, `edit`, `create`.
- Support unicode in slugs if you have international users.
- For public-facing IDs, use opaque/resilient identifiers (SQIDs, ULIDs). Decode the ID portion and ignore the rest of the slug — that way "/posts/abc123-my-old-title" still works after the title changes.
- Check for duplicate route definitions before merging — two routes claiming the same URL is a coin-flip bug.

---

## Things Always Worth Catching

A quick "did I do anything from this list?" pass before opening a PR catches most issues:

1. `Model::create([...])` instead of `new Model()`.
2. Business logic in controllers (a 30-line controller method is a smell).
3. `Auth::user()` in a model.
4. Debug code (`dd`, `dump`, `console.log`).
5. `setTimeout` orchestrating an API call.
6. Arrays passed to Actions instead of DTOs.
7. Direct `response()->json($complexData)` for data that other endpoints also return.
8. Missing table prefix or foreign-key constraint in a migration.
9. `->value` on an enum in PHP code.
10. Manual event listener registration with no comment explaining why.
11. A `Traits/` or `Interface/` folder.
12. Method name that doesn't match what it returns.
13. Editing a published migration.
14. `new Class()` instead of `$this->app->make()` in tests.
15. Overriding core Eloquent methods.
16. Two classes with confusingly similar names.
17. Generic service names (`Service`, `Manager`, `Handler` with no qualifier).
18. Action verbs in REST routes.
19. Mixed underscores/hyphens in config keys.
20. Removing a global scope with no comment.
21. `alert()` in frontend code.
22. Fetching a collection just to call `->count()` on it.
23. Three copies of the same code in three different files.
24. One field encoding two concepts.
25. Forcing type conversions inside a Request class instead of in the Action.
26. Many nullable columns where a JSON column would do.
27. New endpoint duplicating an existing one's purpose.
28. Useless tests (would pass on a broken implementation).
29. `$request->input(...)` instead of `$request->validated()`.
30. Raw `where()` where a scope already exists for that filter.
31. Inline `$request->validate([...])` in a controller instead of a Form Request class.
32. DTO library (Spatie Data, etc.) used as the validator instead of a Form Request.
33. `app(...)` helper (or `new Class()`) to resolve a dependency instead of injecting it.
34. Emoji used as a UI icon instead of an icon-library component or SVG.

If you wrote one of these, fix it before requesting review. "Self-review before requesting review" is the single highest-leverage habit — multiple review rounds with the same class of issue means insufficient self-review, not a strict reviewer.
