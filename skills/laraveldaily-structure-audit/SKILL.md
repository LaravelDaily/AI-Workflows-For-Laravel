---
name: laraveldaily-structure-audit
description: This skill should be used when the user asks to "audit Laravel project structure", "review project architecture", "check code organization", "find over-engineering", "find fat controllers", "structure review", or wants to analyze a Laravel project for structural improvements. Scans all PHP source files and reports findings with actionable suggestions.
---

# Laravel Structure Audit

Analyze a Laravel project's code organization against proven structuring principles. Scan all PHP source files (excluding `vendor/`, `node_modules/`, `storage/`) and produce a structured report of findings with actionable suggestions.

This is NOT a security audit. Focus exclusively on code structure, responsibility placement, and architectural patterns.

## What to Check

### 1. Inline Validation in Controllers

Scan controller methods for `$request->validate(...)`, `Validator::make(...)`, or `$this->validate(...)` calls directly inside controller methods.

**Why it matters:** Inline validation is a normal Laravel pattern for small endpoints, but larger validation blocks become harder to reuse, test, and read when mixed into controllers. Form Request classes are often a better fit when validation grows in size or needs authorization, custom messages, or reuse.

**What to flag:**
- Large inline validation arrays in controller methods (roughly 8+ rules or visibly multi-field business validation)
- `Validator::make(...)` blocks with custom messages, conditional rules, or post-validation hooks living inside controllers
- Similar validation logic repeated across multiple controller methods
- Validation mixed with authorization and business logic in the same controller method
- Ignore simple `$request->validate([...])` in small endpoints and simple `$request->validated()` calls — both can be appropriate

**Suggestion:** When validation is repeated, large, or doing more than basic request checks, extract to a Form Request class using `php artisan make:request`. Move rules to `rules()`, authorization to `authorize()`, and access clean data via `$request->validated()`.

### 2. Fat Controllers (Business Logic in Controllers)

Scan controller methods for signs of business logic that should be extracted:
- Methods longer than ~20 lines of actual logic (excluding blank lines and comments)
- Direct Eloquent queries with complex conditions (multiple `where`, `join`, subqueries)
- Multiple model operations in sequence (create + attach + sync + notify)
- Conditional business rules (if/else chains determining behavior)
- Data transformation or calculation logic

**Why it matters:** Business logic in controllers cannot be reused from Artisan commands, Livewire components, Jobs, or other entry points. Extracting to a Service or Action class enables reuse everywhere.

**What to flag:**
- Controller methods with more than 20 lines of logic
- Controller methods that perform 3+ distinct operations in sequence
- Complex query building directly in controllers
- Business rule conditionals in controllers

**Suggestion:** Extract to a Service class (multiple related methods) or Action class (single operation with `handle()` method). The extracted class can then be called from controllers, commands, Livewire, and Jobs.

**Important — do NOT flag simple CRUD.** A controller method that is just `Model::create($request->validated())` followed by a redirect is fine. Do not suggest extraction for methods under 5 lines or for straightforward create/update/delete with no business rules.

### 3. String Constants for Roles, Statuses, Types

Scan for patterns indicating hardcoded string values used as categories:
- Columns compared against string literals: `$user->role === 'admin'`, `$order->status == 'pending'`
- Arrays of string options: `['admin', 'editor', 'viewer']`
- Validation rules with `in:` containing hardcoded values: `'role' => 'in:admin,editor,user'`
- Constants defined as strings in models: `const ROLE_ADMIN = 'admin';`

**Why it matters:** String comparisons are error-prone (typos compile silently), lack IDE auto-completion, and scatter valid values across the codebase. PHP Enums provide type safety, auto-completion, and a single source of truth.

**What to flag:**
- String comparisons against role/status/type values
- `const` definitions that represent a fixed set of options
- Validation `in:` rules with hardcoded option lists

**Suggestion:** Replace with a PHP Enum backed by string values and cast it in the model using `protected $casts`. Use the Enum in validation with the `Enum` rule.

### 4. Duplicate Logic Across Controllers

Scan for identical or near-identical code blocks appearing in multiple controllers:
- Same query logic copy-pasted across controllers
- Same data transformation in multiple methods
- Same response formatting repeated (especially in API controllers)
- Same authorization/permission check patterns copied across files

**Why it matters:** Duplicate code means duplicate bugs and drift over time. However, premature abstraction is worse than duplication — extract only when the pattern appears in 3 or more places.

**What to flag:**
- Same block of logic (5+ lines) appearing in 3 or more places
- Same query scope logic repeated across controllers instead of being a Model scope
- Same response formatting across API controllers

**Suggestion — depends on what is duplicated:**
- Query logic: Extract to an Eloquent query scope on the Model
- Business logic: Extract to a Service, Action, or Trait
- Response formatting: Extract to a Trait or base controller method
- Only suggest extraction when duplicated 3+ times. Two occurrences are acceptable.

### 5. `env()` Used Outside Config Files

Scan all PHP files outside of `config/` for direct `env()` calls.

**Why it matters:** This is a production bug, not a style preference. After running `php artisan config:cache` (standard in production), `env()` returns `null` everywhere except inside `config/*.php` files. This causes silent failures that only appear in production.

**What to flag:**
- Any `env('...')` call in controllers, models, services, middleware, routes, or any file outside `config/*.php`
- Pay special attention to `env()` in `routes/*.php`, `app/` files, and Blade templates

**Suggestion:** Add a config key in the relevant `config/*.php` file, then use `config('key.name')` everywhere else. For example, replace `env('THIRD_PARTY_API_KEY')` with `config('services.third_party.key')` and define it in `config/services.php`.

### 6. Missing API Resources (Inline Array Transforms)

Scan API controllers for hand-built response arrays instead of API Resource classes.

**Why it matters:** Manually building JSON responses in controllers scatters your API contract across every endpoint. API Resources centralize transformation, are reusable, and handle nested relationships cleanly.

**What to flag:**
- `return response()->json([...])` with hand-built arrays mapping model fields
- `->toArray()` with manual field picking or transformation in controllers
- Returning raw model data that exposes internal columns (timestamps, soft deletes, pivot data) to the API

**Suggestion:** Create API Resources using `php artisan make:resource`. For collections, use `php artisan make:resource --collection`. Especially valuable when the same model is returned from multiple endpoints.

**Exception:** A single endpoint returning a simple `['status' => 'ok']` or a trivial response does not need a Resource class.

### 7. Route Model Binding Not Used

Scan controllers for manual model lookups from route parameters.

**Why it matters:** Route model binding eliminates boilerplate, automatically returns 404 when the model is not found, and makes controller signatures self-documenting.

**What to flag:**
- `Model::find($id)`, `Model::findOrFail($id)`, or `Model::where('id', $id)->firstOrFail()` in controller methods where `$id` comes from a route parameter
- Manual 404 handling after a `find()` call: `if (!$model) abort(404);`

**Suggestion:** Type-hint the model in the controller method signature: `public function show(Post $post)`. For slug-based lookups, override `getRouteKeyName()` on the model. For scoped lookups, use implicit scoping: `public function show(User $user, Post $post)`.

### 8. Authorization Logic Inline Instead of Policies

Scan controllers for manual permission/ownership checks.

**Why it matters:** Authorization scattered across controllers is easy to forget in new endpoints. Policies centralize authorization per model, are auto-discovered in Laravel 11+, and integrate with `Gate`, `@can` in Blade, and `can` middleware.

**What to flag:**
- `if ($user->id !== $post->user_id) abort(403)` or similar ownership checks
- `abort_if`/`abort_unless` with permission-like conditions
- `if (!$user->isAdmin())` type checks in controller methods
- Same authorization logic repeated across multiple controller methods

**Suggestion:** Create a Policy using `php artisan make:policy PostPolicy --model=Post`. Use `Gate::authorize('update', $post)` in controllers, `@can('update', $post)` in Blade, or `->can('update,post')` in route definitions. Policies are auto-discovered in Laravel 11+ — no manual registration needed.

### 9. Route File Bloat and Organization

Scan route files for size, structure, and misuse.

**Why it matters:** In Laravel 11+, `api.php` and `channels.php` don't exist by default — they are opt-in via `php artisan install:api` and `install:broadcasting`. Route files grow silently into unmanageable lists, and it is easy to accidentally mix stateless API behavior into the `web` middleware stack.

**What to flag:**
- `web.php` or `api.php` over ~150 lines with no grouping (`Route::prefix`, `Route::group`, `Route::middleware`)
- Clearly stateless/token-auth API routes living under `web` middleware when they would fit better in `api.php`
- Public or internal JSON endpoints in `web.php` are acceptable if they intentionally rely on session/CSRF/web middleware — do not flag those automatically
- Controllers following resource conventions (index/create/store/show/edit/update/destroy) but routes defined individually instead of using `Route::resource` or `Route::apiResource`
- Dead routes pointing to controllers or methods that don't exist

**Suggestion:** Group related routes with `Route::prefix` and `Route::middleware`. Use `Route::resource` / `Route::apiResource` for resource controllers. If the project has a true stateless API, prefer installing and using `routes/api.php`; otherwise, do not treat JSON-in-`web.php` by itself as a problem. For large applications, split routes into separate files and load them in `bootstrap/app.php`.

### 10. Business Logic in Blade Templates

Scan Blade templates for embedded PHP logic.

**Why it matters:** Blade templates should mainly handle presentation. Querying data, mutating state, or embedding non-trivial business rules in views makes behavior harder to test and reuse. Simple presentation formatting is often fine.

**What to flag:**
- `@php` blocks longer than 2-3 lines
- Eloquent queries in Blade files (`Model::where(...)`, `DB::table(...)`)
- Service container calls, repository/service usage, or other application logic directly from Blade
- Complex ternary chains or multi-line conditionals calculating business values in `{{ }}`
- Mutation or side effects triggered from views
- Repeated markup/behavior across multiple views that should likely become a Blade Component
- Do not flag simple display formatting such as dates, currencies, small string formatting, or straightforward `@if` presentation checks unless the logic becomes hard to read

**Suggestion:** Move queries and business logic to the controller, a View Composer/View creator, an accessor/view model, or a Blade Component class. If the same markup pattern repeats, suggest a Blade Component. Simple display conditionals (`@if ($user->isAdmin())`) and lightweight formatting are fine.

### 11. Scattered Model Lifecycle Logic

Scan for model lifecycle operations (file cleanup, cache invalidation, denormalization, audit logging) that are scattered across controllers and services.

**Why it matters:** When lifecycle logic lives in controllers, it only runs when that specific controller is called. If the model is modified from a different entry point (Artisan command, Job, Livewire, another service), the lifecycle logic is skipped.

**What to flag:**
- File deletion logic (`Storage::delete`) in controller destroy methods that should run whenever the model is deleted
- Cache clearing after model updates in multiple controller methods
- Related model updates (keeping denormalized data in sync) in multiple places
- Audit logging of changes in multiple places

**Suggestion — present options based on complexity:**
- **Model `booted()` closures** — best for simple hooks (1-5 lines). Keeps logic in the model file, visible when you open it, no separate class needed.
- **Observers** — better when lifecycle logic is complex (10+ lines per hook) or when the model file is already large. Use `isDirty('column')` to check what changed and `getRawOriginal('column')` to get old values.
- **Explicit calls in Services/Actions** — most transparent option. The caller sees exactly what happens. Requires discipline to call from every entry point.

Do not prescribe one approach. Present the trade-offs: `booted()` is co-located but can bloat the model; Observers are clean but create hidden side effects; explicit calls are transparent but require discipline.

### 12. Over-Engineering Detection

Scan for signs of unnecessary abstraction layers:
- Action classes that contain only 1-3 lines (just calling `Model::create()`)
- DTO classes wrapping 2-3 fields with no transformation
- Service classes with a single method that is only called from one place
- Pipeline usage for 2 sequential operations that never change
- Event + Listener pairs where only one listener exists AND the listener contains trivial logic (1-3 lines)
- Repository classes that just wrap Eloquent methods without adding query logic
- Interface-for-everything: `UserRepositoryInterface` / `UserServiceInterface` when there is only one implementation and no realistic second one. Interfaces earn their place when crossing package/module boundaries or when multiple implementations exist.
- Multiple User-like models (`Admin.php`, `Manager.php`, `Staff.php`) extending `Authenticatable` alongside `User.php` — use a single `User` model with a role column and Policies instead

**Why it matters:** Every extra class is a file to maintain, a level of indirection to follow, and cognitive overhead for the team. Abstraction should earn its place by enabling reuse, testability, or clarity — not exist as ceremony.

**What to flag:**
- Action classes under 5 lines of logic in `handle()`/`execute()`
- DTO classes with fewer than 4 properties and no transformation logic
- Service classes with one method used from one place
- Repository classes that mirror Eloquent API without adding query logic (Eloquent IS the data access layer in Laravel — a Repository on top is a second ORM)
- Interfaces with only one implementation and no module/package boundary justification
- Multiple `Authenticatable` models for what should be roles on a single `User` model

**Suggestion:** Inline the logic back into the caller. Simple CRUD does not need Action/DTO/Service layers. Extract only when complexity or reuse justifies it. For single-implementation interfaces, remove the interface and depend on the concrete class directly. For multiple user models, consolidate to a single `User` model with a `role` column (PHP Enum) and use Policies/middleware for authorization.

**Exception:** Do not flag multiple `Authenticatable` models in multi-tenancy setups or projects with truly separate authentication domains (different databases). Do not flag consistent use of Actions/Services across a project if the team clearly values uniformity — consistency has its own value.

## Output Format

Present findings in this structure:

### Summary

A brief overview: total findings count, top 2-3 most impactful areas to address.

### Findings by Category

For each category that has findings:

**Category Name**

| Severity | File | Line(s) | Issue | Suggestion |
|----------|------|---------|-------|------------|
| High/Medium/Low | path/to/file.php | 42-58 | What was found | What to do |

### Severity Levels

- **High** — Clear production bugs or strong architectural hazards: `env()` outside config, dead routes, stateless API behavior accidentally relying on `web` middleware, repeated lifecycle logic causing inconsistent side effects
- **Medium** — Meaningful structural issues that are hurting clarity, reuse, or maintainability: large/frequently repeated inline validation, genuinely fat controllers, duplicate logic starting to drift, inline authorization spread across endpoints, route file bloat, heavy business logic in Blade
- **Low** — Context-dependent improvements or local cleanup: missing route model binding in a few spots, small over-engineering, minor API Resource opportunities, could-be-enum constants in only one area, simple formatting logic that might read better elsewhere

### What's Done Well

End with a short section acknowledging patterns the project already follows correctly. This prevents the audit from feeling like a list of complaints and validates good decisions.

## Important Guidelines

- Do NOT nitpick small files or simple CRUD. A 5-line controller method is fine as-is.
- Do NOT suggest extraction for code used in only one place unless it is genuinely complex (20+ lines of logic).
- Do NOT flag architectural choices that are clearly intentional and consistent (e.g., the project consistently uses Actions everywhere — don't flag the simple ones if the team values consistency).
- DO read enough of each file to understand context before flagging. A long method might be long because it contains comments, not logic.
- DO consider the project size. A 5-controller CRUD app does not need Services, Actions, and DTOs.
- DO check which Laravel version the project uses (`composer.json`) — some suggestions only apply to Laravel 11+ (auto-discovery of Policies, slim skeleton, `install:api`).
- Present findings as suggestions, not mandates. Structure is personal preference — the audit surfaces opportunities, not requirements.
