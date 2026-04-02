---
name: laraveldaily-testing-audit
description: This skill should be used when the user asks to "audit Laravel tests", "review test coverage", "check testing practices", "find missing tests", "testing review", "improve tests", or wants to analyze a Laravel project's testing quality and identify gaps. Scans existing tests and application code, then reports findings with actionable suggestions.
---

# Laravel Testing Audit

Analyze a Laravel project's test suite against proven testing practices. Scan all PHP source files in `tests/`, `app/`, and `routes/`, then produce a structured report of findings with actionable suggestions for improving test quality and coverage where it matters most.

This is NOT a code quality or structure audit. Focus exclusively on testing: what is tested, what is missing, how tests are written, and whether the test suite protects the project's critical paths.

## Before You Start

1. **Detect the test framework.** Check `composer.json` for `pestphp/pest` or `phpunit/phpunit`. Check whether `tests/Pest.php` exists. This determines the syntax used in suggestions.
2. **Detect the Laravel version** from `composer.json` — some testing features differ across versions.
3. **Scan `tests/`** to understand what already exists before flagging gaps.
4. **Scan `app/` and `routes/`** to understand what the application does.

## What to Check

### 1. Missing Tests for Critical Paths

Scan the application for critical features that have no corresponding tests.

**Why it matters:** The real goal of testing is NOT 100% code coverage — it is protecting features that would cause serious damage if broken. A project with 30% coverage on critical paths is better tested than one with 90% coverage on trivial getters.

**Three categories of critical features to check (in priority order):**

1. **Main pages and endpoints** — home page, primary listing pages, core API endpoints. These are what users see first. At minimum, smoke tests (assert HTTP 200) should exist.
2. **Authentication and authorization** — login, registration, password reset, role-based access, middleware-protected routes. A broken auth flow locks out every user or worse, lets everyone in.
3. **Payment and financial transactions** — if the project has any payment processing, billing, subscription, or financial logic, it must have tests. A bug here causes direct monetary loss.

**What to flag:**
- Routes with `auth` or custom authorization middleware that have zero test coverage
- Controller methods handling money, payments, or subscriptions with no tests
- Public-facing pages (home, dashboard, main listings) with no smoke tests
- API endpoints with no tests at all
- Form submissions (POST/PUT/DELETE routes) with no tests

**Suggestion:** Start with smoke tests (`$response->assertOk()`) for every critical route. Then add targeted tests for authentication gates, form submissions, and any financial logic. Prioritize by impact — what would cause the most damage if broken?

### 2. Only Happy Path Tested (Missing Sad Path)

Scan existing tests for patterns that only test the successful scenario.

**Why it matters:** Happy path tests verify that things work when everything goes right. But users submit empty forms, access pages they shouldn't, and send malformed data. Sad path tests catch the bugs that actually ship to production.

**What to flag:**
- Test files that only have `assertOk()` / `assertSuccessful()` / `assertCreated()` with no corresponding failure assertions
- Form/API submission tests that send valid data but never test with invalid/empty data
- Tests for authenticated routes that only test with a logged-in user, never as a guest
- Tests for authorized actions that only test with an admin, never with a regular user
- Tests that create records but never test what happens with duplicate or constraint-violating data

**Suggestion:** For every happy path test, consider what can go wrong:
- Submit empty or invalid data — assert `assertInvalid()`, `assertSessionHasErrors()`, or `assertUnprocessable()` (422)
- Access as guest — assert `assertRedirect(route('login'))` or `assertUnauthorized()` (401)
- Access without permission — assert `assertForbidden()` (403)
- Request a non-existent resource — assert `assertNotFound()` (404)

### 3. Hardcoded Test Data Instead of Factories

Scan test files for manually constructed data arrays and direct `Model::create()` calls.

**Why it matters:** Hardcoded test data is brittle — when a migration adds a required column, every hardcoded array in the test suite breaks. Factories centralize fake data generation, scale effortlessly, and produce realistic values.

**What to flag:**
- `Model::create([...])` with inline field arrays in test files
- `new Model([...])` with hardcoded field values in tests
- Large arrays of fake data defined directly in test methods
- Test data that uses unrealistic values like `'test'`, `'abc'`, `'123'` for fields that have semantic meaning
- Models that lack a corresponding Factory class entirely

**Suggestion:** Create Factories using `php artisan make:factory`. Use `Model::factory()->create()` in tests. Override only the fields relevant to the test: `Product::factory()->create(['price' => 0])`. This makes tests resilient to schema changes and clearly communicates which fields matter for each test case.

### 4. Repeated Setup Code Across Tests

Scan test files for duplicated arrangement logic.

**Why it matters:** When the same setup (creating users, seeding data, authenticating) is repeated in every test method, tests become verbose and harder to maintain. A single change to the setup requires editing dozens of test methods.

**What to flag:**
- Same `actingAs(User::factory()->create())` call repeated in multiple tests within the same file
- Same model creation logic duplicated across test methods
- Same data setup pattern appearing in 3+ test methods in one file
- Test files longer than 200 lines with obvious repetition

**Suggestion (depends on framework):**
- **Pest:** Use `beforeEach()` hooks to set up shared state. Store in `$this->user`, `$this->product`, etc. Create custom helper functions in `tests/Pest.php` (e.g., `function asAdmin()` that creates and authenticates an admin user).
- **PHPUnit:** Use `setUp()` method with `parent::setUp()` call. Store shared objects as class properties.

### 5. Missing Database Isolation

Scan test configuration and test files for database setup issues.

**Why it matters:** Tests that share database state between runs produce flaky results — a test passes alone but fails when run with the suite, or worse, tests pass locally but fail in CI. Running tests against a live database risks data loss.

**What to flag:**
- Test classes missing both `RefreshDatabase` and `DatabaseTransactions` traits
- `phpunit.xml` or `.env.testing` not configured for a test database (no SQLite in-memory, no separate DB name)
- Tests that depend on specific database state from previous tests (order-dependent tests)
- Tests that seed the production database or reference `.env` database directly
- Test files that use `DB::table()->insert()` without any database reset mechanism

**Suggestion:** Add `use RefreshDatabase;` (Pest: `uses(RefreshDatabase::class)`) to every test file that touches the database. Configure `phpunit.xml` with `<env name="DB_CONNECTION" value="sqlite"/>` and `<env name="DB_DATABASE" value=":memory:"/>` for fast, isolated test runs. For tests needing MySQL-specific features, use a dedicated test database via `.env.testing`.

### 6. Side Effects Not Faked (Mail, Notifications, Queue, Storage, HTTP)

Scan for tests that trigger side effects without using Laravel's fake utilities.

**Why it matters:** Tests should never send real emails, push real jobs, write real files, or hit real external APIs. Beyond being slow and unreliable, this can send test emails to real users, charge real credit cards, or spam external services.

**What to flag:**
- Test files that call routes/methods known to send Mail, Notifications, or dispatch Jobs, but do not call `Mail::fake()`, `Notification::fake()`, or `Queue::fake()` respectively
- Tests involving file uploads without `Storage::fake()`
- Tests calling external APIs without `Http::fake()`
- Controller methods that trigger side effects where the corresponding test file has no fake assertions

**Suggestion:** Follow the three-step fake pattern:
1. Call `Mail::fake()` / `Notification::fake()` / `Queue::fake()` / `Storage::fake()` / `Http::fake()` **before** the action
2. Execute the action being tested
3. Assert the side effect: `Mail::assertSent(OrderConfirmation::class)`, `Notification::assertSentTo($user, ...)`, `Queue::assertPushed(ProcessPayment::class)`, `Storage::assertExists('uploads/file.pdf')`, `Http::assertSent(...)`

### 7. Fragile Assertions (assertSee Misuse)

Scan test files for assertions that are prone to false positives or false negatives.

**Why it matters:** `assertSee('Product Name')` passes if that text appears ANYWHERE on the page — in the nav bar, footer, a different product, or an HTML attribute. This creates false confidence: the test passes even when the feature is broken.

**What to flag:**
- `assertSee()` used as the primary assertion for data presence on pages (especially with common words or short strings)
- `assertSee()` used to verify form field values or specific data display
- Tests that only check HTTP status codes without verifying response content or database state
- Tests that assert on raw HTML structure which breaks on any template change

**Suggestion:** Prefer `assertViewHas()` to verify actual data passed to views. Use `assertSee()` only for unique, specific strings. For API tests, use `assertJson()` to check response structure. Combine status assertions with data assertions: check both that the response is 200 AND that the correct data is present via `assertViewHas()` or `assertDatabaseHas()`.

### 8. Testing Package Internals Instead of Integration

Scan test files for tests that re-test functionality provided by well-tested packages or the framework itself.

**Why it matters:** Laravel's framework and reputable packages have their own comprehensive test suites. Re-testing that `auth` middleware redirects guests, or that Eloquent `save()` persists data, wastes time without catching real bugs. Test YOUR code's integration with these packages, not the packages themselves.

**What to flag:**
- Tests that verify basic Eloquent operations (create, find, delete) without any custom business logic
- Tests that just verify a middleware exists on a route without testing the actual user flow
- Tests that verify framework features like CSRF protection, session handling, or cache drivers
- Unit tests for trivial model accessors, mutators, or relationships that have no custom logic

**Suggestion:** Focus tests on YOUR application logic that uses these packages. Instead of testing that "auth middleware redirects to login" (the framework guarantees this), test that "a guest cannot access the admin dashboard and is redirected to login" — this tests your route configuration and authorization setup. Think of packages as LEGO bricks tested by their makers — you test the house you build with them.

### 9. No Architecture or Quality Gate Tests

Check if the project uses Pest architecture testing or equivalent quality gates.

**Why it matters:** Architecture tests catch structural problems before they reach production: leftover `dd()` / `dump()` calls, models not extending the base class, controllers using non-REST methods, and other anti-patterns. They act as automated code review.

**What to flag:**
- No `arch()` tests in the test suite (Pest projects)
- No automated checks preventing `dd()`, `dump()`, `ray()`, or `ddd()` from being committed
- No enforcement of class structure conventions (models extend Model, controllers in correct namespace, etc.)
- Projects using Pest that don't leverage architecture presets

**Suggestion:** Add architecture tests in a dedicated test file (e.g., `tests/ArchTest.php`):
```php
arch('no debugging calls')
    ->expect(['dd', 'dump', 'ray', 'ddd'])
    ->not->toBeUsed();

arch('models extend base model')
    ->expect('App\Models')
    ->toExtend('Illuminate\Database\Eloquent\Model');
```
Pest provides built-in presets: `arch()->preset()->laravel()` enforces Laravel conventions including REST controller methods. Use `arch()->preset()->security()` to catch security anti-patterns.

### 10. Poor Test Organization and Naming

Scan test file structure and test descriptions for clarity issues.

**Why it matters:** When a test fails in CI, the developer sees only the test name. A test named `test_it_works` tells nothing. Good test names describe the scenario and expected outcome, making failures immediately actionable.

**What to flag:**
- Test methods/descriptions with vague names: `test_it_works`, `test_basic`, `it('works')`, `it('returns correct response')`
- Test files not following a clear naming convention (no `Test` suffix, inconsistent naming)
- All tests in a single file instead of organized by feature or model
- Feature tests and Unit tests mixed inappropriately (unit tests hitting the database, feature tests not using HTTP methods)
- No use of Pest groups or PHPUnit `@group` annotations in large test suites

**Suggestion:** Name tests to describe the scenario: `it('prevents guests from accessing admin dashboard')`, `it('validates that product name is required')`, `it('sends confirmation email after order is placed')`. Organize tests by feature: `tests/Feature/ProductTest.php`, `tests/Feature/Auth/LoginTest.php`. In large suites, use Pest `->group('api', 'products')` or PHPUnit `@group` to enable selective test execution.

### 11. Missing Validation Testing

Scan for form endpoints and API endpoints that have validation rules but no corresponding validation tests.

**Why it matters:** Validation is the first line of defense against bad data. If validation rules exist but are never tested, they can be accidentally removed or weakened during refactoring without anyone noticing.

**What to flag:**
- Form Request classes (extending `FormRequest`) with rules that have no test asserting validation behavior
- Controller methods with `$request->validate([...])` where no test sends invalid data
- API endpoints that accept POST/PUT data with no test for the 422 response
- Validation rules with complex conditional logic (`required_if`, `required_with`, custom rules) that are never tested

**Suggestion:** For each form or API endpoint, add at least one test that submits invalid data and asserts the expected validation errors:
```php
// Pest example
it('requires a product name', function () {
    $response = actingAs($this->admin)
        ->post('/products', ['name' => '', 'price' => 10]);
    $response->assertInvalid(['name']);
});
```
Use `assertInvalid(['field'])` to verify specific fields fail validation. Use `assertValid()` after successful submissions to confirm no unexpected validation errors.

### 12. Debug Code Left in Tests

Scan test files for debugging statements that should not be committed.

**Why it matters:** `dd()`, `dump()`, `$response->dump()`, `$response->dumpHeaders()`, and `withoutExceptionHandling()` are valuable debugging tools during development but should not remain in committed tests. `withoutExceptionHandling()` in particular changes test behavior by preventing Laravel from converting exceptions to HTTP responses, which can mask real bugs.

**What to flag:**
- `dd(...)` or `dump(...)` calls in test files
- `$response->dump()` or `$response->dumpHeaders()` calls
- `ray(...)` or `ddd(...)` calls
- `withoutExceptionHandling()` left in committed tests (unless intentionally testing exception classes)
- Commented-out assertions or test logic

**Suggestion:** Remove all debugging calls before committing. If `withoutExceptionHandling()` is needed for a specific test scenario (testing that a specific exception is thrown), add a comment explaining why. Better yet, enforce this with a Pest architecture test: `arch()->expect(['dd', 'dump', 'ray'])->not->toBeUsed()`.

## Output Format

Present findings in this structure:

### Summary

A brief overview: what the test suite covers, total findings count, and top 2-3 most impactful areas to address.

### Findings by Category

For each category that has findings:

**Category Name**

| Severity | File | Line(s) | Issue | Suggestion |
|----------|------|---------|-------|------------|
| High/Medium/Low | path/to/file.php | 42-58 | What was found | What to do |

### Severity Levels

- **High** — Critical gaps that leave the application vulnerable: no tests for authentication/authorization flows, missing database isolation (tests hitting production DB), side effects not faked (real emails sent during tests), no tests for payment/financial logic
- **Medium** — Meaningful testing gaps that reduce confidence: only happy paths tested, missing validation tests for important forms, fragile `assertSee()` assertions on critical pages, hardcoded test data making tests brittle, no architecture tests preventing debug code in production
- **Low** — Improvements that would make the test suite better but are not urgent: repeated setup code, vague test names, minor organization issues, testing package internals, missing groups in large suites

### What's Done Well

End with a short section acknowledging what the test suite already does correctly. This validates good decisions and prevents the audit from feeling like a list of complaints.

## Important Guidelines

- Do NOT demand 100% code coverage. Focus on critical paths first.
- Do NOT flag every route without a test. Prioritize by risk and impact.
- Do NOT insist on unit tests for simple CRUD. Feature tests covering the full flow are often sufficient and more valuable.
- Do NOT suggest testing framework or package internals.
- DO consider the project size. A small 5-controller app needs fewer tests than a large SaaS platform.
- DO check the test framework (Pest vs PHPUnit) and tailor suggestions to the syntax the project uses.
- DO check for `.env.testing` and `phpunit.xml` configuration as part of the audit.
- DO prioritize findings. Authentication and payment tests missing is more important than vague test names.
- Present findings as suggestions, not mandates. Testing strategy involves trade-offs — the audit surfaces opportunities, not requirements.
