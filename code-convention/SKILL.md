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

---

## DTOs and Request Validation

### Use DTOs (Spatie Laravel Data), not arrays, for Action inputs

Use the `Data` suffix (e.g., `StudentProfileData extends Data`). Passing arrays into Actions defeats the whole point: you lose types, lose IDE support, lose validation at the boundary, and gain nothing.

### Always use validated data, never raw input

```php
// PREFERRED — Form Request
public function store(StoreCommentRequest $request, StoreCommentAction $action): RedirectResponse
{
    $validatedData = $request->validated();
    $action->execute($validatedData);
    return redirect()->back();
}

// PREFERRED — inline validation
public function store(Request $request, StoreCommentAction $action): RedirectResponse
{
    $validatedData = $request->validate([
        'text' => ['required', 'string', 'max:500'],
    ]);
    $action->execute($validatedData);
    return redirect()->back();
}

// AVOID
$action->execute($request->input('text'));
```

`$request->input()` returns whatever the client sent, including fields you didn't ask for and types you didn't expect. `validated()` returns only the allow-listed, cast values.

### Don't force type conversions inside the Request class

Request classes describe *what's valid*. If you need to transform a string into an enum, parse a date, or hydrate a DTO, that belongs in the Action — where the conversion is testable and reusable.

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

If you wrote one of these, fix it before requesting review. "Self-review before requesting review" is the single highest-leverage habit — multiple review rounds with the same class of issue means insufficient self-review, not a strict reviewer.
