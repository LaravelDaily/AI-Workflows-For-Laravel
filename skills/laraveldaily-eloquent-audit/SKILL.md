---
name: laraveldaily-eloquent-audit
description: This skill should be used when the user asks to "audit Eloquent code", "review Eloquent usage", "check model code", "find N+1 queries", "find query performance issues", "Eloquent review", or wants to analyze a Laravel project's Eloquent usage for improvements. Scans relevant Laravel source files and reports findings with actionable suggestions.
---

# Laravel Eloquent Audit

Analyze a Laravel project's Eloquent usage against proven patterns and performance practices. Scan relevant PHP source files plus migrations, model-related tests, and API/resource code (excluding `vendor/`, `node_modules/`, `storage/`) and produce a structured report of findings with actionable suggestions.

This is NOT a security audit or a full application review. Focus exclusively on Eloquent model configuration, query patterns, relationship usage, serialization, write correctness, and performance.

## What to Check

### 1. N+1 Query Problems

Scan controllers, Livewire components, services, and Blade templates for relationship access without eager loading.

**Why it matters:** The N+1 problem is the most common Eloquent performance issue. Displaying 30 posts with their authors causes 31 queries (1 for posts + 30 for each author) instead of 2. This multiplies with nested relationships — 30 posts with authors and comments can cause hundreds of queries. The difference between 2 queries and 200 queries is the difference between a fast page and a timeout.

**What to flag:**
- Queries fetching collections (`all()`, `get()`, `paginate()`) without `with()` when relationships are accessed in the view or downstream code
- Relationship access inside `@foreach` loops in Blade templates where the parent query has no eager loading
- Accessing relationships inside accessors — this is a hidden N+1 because the accessor runs per model instance
- Using `->count()` on a relationship method (`$user->posts()->count()`) inside a loop instead of `withCount()` or using the loaded collection `$user->posts->count()`
- Packages like Spatie Media Library relationships not being eager loaded (`->with('media')`)
- Do NOT flag single model lookups (`find()`, `first()`) where only one relationship is accessed — that is 2 queries total, not N+1

**Suggestion:** Add `->with('relationship')` to the query. For counts, use `->withCount('relationship')` which adds a `{relation}_count` attribute via subquery. For aggregates, use `withSum()`, `withAvg()`, `withMin()`, `withMax()`. Consider enabling `Model::preventLazyLoading(!app()->isProduction())` in `AppServiceProvider` to catch N+1 issues during development.

### 2. Filtering in PHP Instead of Database

Scan for patterns where data is loaded into PHP collections and then filtered, instead of using database-level `where()` clauses.

**Why it matters:** Loading all records then filtering with collection methods (`filter()`, `reject()`, `where()` on collections) wastes memory and database bandwidth. A query returning 4000 records filtered to 10 in PHP can use dramatically more memory than a database query returning 10 records directly. The database is almost always better at filtering than PHP.

**What to flag:**
- `Model::all()` or `->get()` followed by `->filter()`, `->reject()`, `->where()` on the collection
- Loading full collections then using `->first()` on the collection instead of `->first()` on the query
- `->get()` followed by `->take()`, `->slice()`, or `->skip()` instead of using `limit()` and `offset()` in the query
- Search/filter features that load all records then search in PHP
- Do NOT flag collection methods used on already-small, bounded datasets (e.g., filtering a user's 3 roles)

**Suggestion:** Move filtering to the query builder with `where()`, `when()`, `whereHas()`, `whereRelation()`. Use `when()` for conditional filters: `->when($request->search, fn($q, $s) => $q->where('title', 'like', "%$s%"))`. Use `limit()` / `take()` at the query level instead of slicing collections.

### 3. Large Datasets Loaded Into Memory

Scan commands, jobs, imports, exports, and maintenance scripts for `get()`/`all()` usage on large tables where chunking or streaming would be safer.

**Why it matters:** The most expensive Eloquent problems are not always query count. Loading 50,000 or 500,000 rows into memory at once can crash workers, exhaust PHP memory, or make simple maintenance tasks unusably slow. For large data processing, iteration strategy matters as much as the query itself.

**What to flag:**
- `Model::all()` or `->get()` used in jobs, commands, or imports that appear to process entire tables or very large subsets
- Long-running loops over large result sets without `chunk()`, `chunkById()`, `lazy()`, or `cursor()`
- Batch update/cleanup scripts that hydrate full models when only IDs or a few columns are needed
- Offset-based pagination over changing large datasets where `chunkById()` would be safer
- Do NOT flag ordinary paginated controller actions or clearly small bounded datasets

**Suggestion:** Use `chunk()` for bounded batch processing, `chunkById()` when rows may be inserted/deleted during iteration, `lazy()` for memory-friendly streaming with model hydration, and `cursor()` when you truly need one-at-a-time traversal. Prefer selecting only the columns needed for the batch operation.

### 4. Missing Column Selection (SELECT *)

Scan queries, especially in API endpoints, for loading all columns when only a few are needed.

**Why it matters:** `SELECT *` transfers every column from the database, including large text fields, JSON blobs, and columns never used in the response. In API endpoints this directly inflates response payloads. Even for web views, selecting fewer columns reduces memory usage and speeds up hydration.

**What to flag:**
- API controller methods returning models or collections without `select()` — especially when the model has many columns or large text/JSON fields
- Eager-loaded relationships without column constraints (`->with('user')` instead of `->with('user:id,name,email')`)
- `Model::all()` in API responses
- Missing foreign keys or primary keys in `select()` calls that would break relationships
- Do NOT flag admin panels, simple CRUD, or internal tools where payload size is not critical

**Suggestion:** Use `select()` on queries: `Post::select(['id', 'title', 'user_id'])`. Constrain eager-loaded relationships: `->with('user:id,name')`. Always include the primary key and any foreign keys needed for relationships. For API endpoints, prefer API Resources (`php artisan make:resource`) to control response shape consistently.

### 5. Missing or Incorrect Model Casts

Scan models for columns that should be cast but are not.

**Why it matters:** Without proper casts, date columns return strings instead of Carbon instances, boolean columns return integers, JSON columns return raw strings, and enum columns return plain strings instead of type-safe PHP Enums. Missing casts cause subtle bugs that only surface in edge cases.

**What to flag:**
- Date/datetime columns not cast to `'datetime'` — look at migration files for `timestamp`, `date`, `dateTime` columns and verify corresponding casts exist
- Boolean columns (`tinyint(1)`, `boolean` in migrations) not cast to `'boolean'`
- JSON columns (`json` in migrations) not cast to `'array'`, `'collection'`, or `AsCollection::class`
- Columns representing a fixed set of values (status, role, type) not cast to a PHP Enum when the project uses PHP 8.1+
- Do NOT flag legacy projects that intentionally avoid casts for compatibility
- Do NOT treat old-but-valid `$casts` property syntax as a problem by itself

**Suggestion:** Define appropriate casts for columns whose runtime type matters: dates, booleans, JSON, and enums. On Laravel 11+, the `casts()` method is the modern style; older `$casts` property syntax is still valid. For status/role/type columns, create a backed PHP Enum and cast to it for type safety and IDE auto-completion when that matches the project's conventions.

### 6. Hardcoded String Comparisons for Statuses, Roles, Types

Scan for string literal comparisons against model columns that represent a finite set of values.

**Why it matters:** Comparing `$user->role === 'admin'` throughout the codebase is error-prone. String values lack IDE auto-completion, are harder to refactor safely, and scatter the list of valid values across the codebase. PHP Enums solve all of these problems.

**What to flag:**
- String comparisons against role/status/type columns: `$model->status === 'pending'`, `$order->type == 'subscription'`
- Arrays of string options: `['pending', 'active', 'cancelled']`
- Validation `in:` rules with hardcoded option lists: `'status' => 'in:pending,active,cancelled'`
- Model constants representing options: `const STATUS_ACTIVE = 'active'`
- Do NOT flag one-off string comparisons against non-categorical data

**Suggestion:** Create a backed PHP Enum: `enum OrderStatus: string { case Pending = 'pending'; case Active = 'active'; }`. Cast it on the model: `'status' => OrderStatus::class`. Use in validation: `'status' => [new Enum(OrderStatus::class)]`. Compare with type safety: `$order->status === OrderStatus::Active`.

### 7. Mass Assignment Risks or Silent Failures

Scan models for missing or incorrect `$fillable`/`$guarded` configuration, and check whether strict mode is enabled.

**Why it matters:** Mass assignment problems are often silent. Attributes are discarded with no obvious error, or user input flows into unguarded models in ways the developer did not intend. The risk is not the presence of one style or another; the risk is mismatched model guarding and real write paths.

**What to flag:**
- Models with neither `$fillable` nor `$guarded` defined that are used with `create()`, `update()`, or `fill()` arrays
- Models with `$guarded = []` (fully unguarded) that are written from request data or other user-controlled input without clear validation/DTO boundaries
- `$fillable` arrays that are clearly out of sync with actual write paths, causing attributes to be silently ignored
- No `Model::shouldBeStrict()` call in `AppServiceProvider` — this would catch silent mass assignment failures during development
- Do NOT flag `$guarded = []` in projects that consistently validate and shape input before mass assignment
- Do NOT insist that every model use `$fillable` specifically — assess the safety of the actual pattern in use

**Suggestion:** Align model guarding with real write paths. Use `$fillable`, `$guarded`, DTOs, or validated arrays consistently, but avoid silent discard scenarios. Enable strict mode in development: `Model::shouldBeStrict(!app()->isProduction())` in `AppServiceProvider::boot()`. This makes Eloquent throw exceptions for mass assignment violations, lazy loading, and accessing missing attributes — catching bugs early instead of silently ignoring them.

### 8. Transactions and Multi-Step Write Correctness

Scan services, actions, controllers, commands, and jobs for multi-step Eloquent writes that should likely be wrapped in a database transaction.

**Why it matters:** Creating a record, attaching relations, updating counters, and writing audit rows across multiple statements is common in Laravel apps. Without a transaction, a failure halfway through leaves partial state behind. This is a correctness issue, not just style.

**What to flag:**
- Multi-step create/update/delete flows that must succeed or fail as a unit but have no `DB::transaction()`
- Code paths that update several related models in sequence with no rollback strategy
- Inventory, balance, quota, or status transitions that can leave inconsistent state if one later query fails
- Do NOT flag a single `create()` or simple isolated update that stands on its own

**Suggestion:** Wrap genuinely atomic write flows in `DB::transaction()`. Keep the transaction scope tight and include all writes that must stay consistent together. If there are follow-up side effects such as events or jobs, make sure they happen after commit when correctness depends on committed data.

### 9. Convenience Methods Used Without Correctness Guarantees

Scan for `firstOrCreate()`, `updateOrCreate()`, and `upsert()` usage that assumes these methods alone prevent duplicates or race conditions.

**Why it matters:** These helpers are convenient, but they do not replace database constraints. Under concurrent requests, two workers can still attempt the same insert unless the database enforces uniqueness. This is a classic source of duplicate rows and subtle production-only bugs.

**What to flag:**
- `firstOrCreate()` or `updateOrCreate()` used for uniqueness-sensitive data with no matching unique index visible in migrations
- Code that assumes these helpers are race-condition-safe by themselves
- `upsert()` usage where the conflict target or unique columns do not appear to match the intended business rule
- Do NOT flag convenience helpers used on low-risk data where duplicates are harmless

**Suggestion:** Keep the helper if it reads well, but back it with a real unique index in the database. For truly atomic uniqueness guarantees, rely on schema constraints first and application helpers second. Use transactions where surrounding writes must stay consistent with the upserted row.

### 10. Repeated Query Logic Not Extracted to Scopes

Scan for identical or near-identical query conditions appearing in multiple places.

**Why it matters:** When the same `where()` chain appears in 3+ controllers or services, it is duplicated logic that drifts over time. Local scopes centralize query conditions on the model, making them reusable, testable, and self-documenting. A scope named `verified()` is clearer than `whereNotNull('email_verified_at')` repeated everywhere.

**What to flag:**
- The same `where()` condition (or combination of conditions) appearing in 3 or more places across different files
- Complex multi-condition filters copy-pasted across controllers
- Date range filters, status filters, or role checks repeated in multiple queries
- Do NOT flag conditions used in only 1-2 places — extraction at that point is premature
- Do NOT flag simple, obvious conditions like `where('id', $id)`

**Suggestion:** Extract repeated query conditions into a local scope on the model. On newer Laravel versions, `#[Scope]` is a modern option; older `scopeXxx()` methods remain valid. Choose the style that matches the project. Reserve global scopes for truly universal filters like multi-tenancy.

### 11. Sensitive Data Exposed in Serialization

Scan models for missing `$hidden` configuration and API responses that expose internal columns.

**Why it matters:** When models are serialized to JSON, every column is included by default. This exposes `password`, `remember_token`, `two_factor_secret`, internal flags, soft delete timestamps, and pivot data to API consumers. Even if the frontend ignores these fields, they are still exposed.

**What to flag:**
- User/auth models without `$hidden` including at minimum `password` and `remember_token`
- Models with sensitive columns (tokens, secrets, internal flags) not listed in `$hidden`
- API endpoints returning raw models without API Resources, where internal columns are exposed
- `$appends` including computed attributes that expose internal logic or trigger additional queries
- Do NOT flag internal admin tools or CLI output where exposure is not a concern

**Suggestion:** Add `protected $hidden = ['password', 'remember_token']` to auth models. For broader control, use `$visible` to whitelist only the fields that should be serialized. For API endpoints, use API Resources (`php artisan make:resource`) to explicitly control the response shape. Use `makeHidden()` and `makeVisible()` for per-response adjustments.

### 12. Accessors Causing Hidden Performance Issues

Scan accessors for relationship loading, database queries, or expensive computations.

**Why it matters:** Accessors run once per model instance. An accessor that loads a relationship or runs a query causes N+1 problems that are invisible. A seemingly simple accessor can trigger a query per model in a collection loop.

**What to flag:**
- Accessors that access relationships not guaranteed to be eager loaded (e.g., `$this->networks`, `$this->posts`)
- Accessors containing `DB::`, `Model::where()`, or any database query
- Accessors performing expensive computations when used in collection contexts
- `$appends` including accessors that trigger queries — these run on every serialization
- Do NOT flag simple formatting accessors (date formatting, string concatenation, basic calculations)

**Suggestion:** Move query-dependent logic out of accessors into the controller or service layer. If the accessor needs relationship data, ensure it is always eager loaded via `$with` or explicit `with()`. For aggregates, use `withCount()`, `withSum()`, etc. at query time instead of computing in accessors. If the value is expensive to compute and rarely changes, consider caching it as a database column.

### 13. Inefficient Relationship Patterns

Scan for relationship usage patterns that have simpler or more performant Eloquent alternatives.

**Why it matters:** Laravel offers specialized relationship methods that reduce query count and code complexity. Using `whereHas()` + `with()` with the same closure is duplicated logic that `withWhereHas()` solves in one call. Counting with `$user->posts()->count()` runs a query per user when `withCount('posts')` does it in one subquery. Loading all related records to find the latest one wastes memory when `latestOfMany()` returns a single record.

**What to flag:**
- `whereHas()` and `with()` using the same closure — replace with `withWhereHas()`
- `->count()` called on relationship methods inside loops — replace with `withCount()`
- Loading a full `hasMany` relationship then accessing only the first/last record — use `latestOfMany()`, `oldestOfMany()`, or `ofMany()`
- Manual `where('user_id', $user->id)` instead of `whereBelongsTo($user)`
- Deeply nested eager loading (`with('projects.tasks')`) when only the deepest level is needed — consider `hasManyThrough()`
- Do NOT flag patterns where the simpler alternative does not exist for the Laravel version in use

**Suggestion:** Use `withWhereHas()` to combine filtering and eager loading in one call. Use `withCount()` / `withSum()` / `withAvg()` for aggregates instead of loading full relationships. Define `latestOfMany()` / `oldestOfMany()` relationships for single-record access from has-many. Use `whereBelongsTo($model)` for cleaner belongs-to filtering.

### 14. Missing Cleanup for Old Records

Scan for tables that accumulate records indefinitely without any pruning strategy.

**Why it matters:** Tables like `password_reset_tokens`, `sessions`, `activity_logs`, `notifications`, `failed_jobs`, and `job_batches` grow indefinitely in production. Over time they slow down queries, bloat backups, and waste storage. Laravel's `Prunable` and `MassPrunable` traits provide a built-in solution.

**What to flag:**
- Models for inherently temporary data (tokens, logs, notifications, sessions) with no `Prunable` or `MassPrunable` trait
- No `model:prune` command in the scheduler (`routes/console.php` or `app/Console/Kernel.php`)
- Models using `Prunable` for high-volume tables where `MassPrunable` would be more appropriate
- `Prunable` models with associated files or external resources but no `pruning()` hook to clean them up
- Do NOT flag core business data tables — only flag tables that are clearly accumulating disposable records

**Suggestion:** Add `use Prunable` (or `MassPrunable` for high-volume tables) to the model and define the `prunable()` method: `public function prunable(): Builder { return static::where('created_at', '<', now()->subDays(30)); }`. Schedule it: `Schedule::command('model:prune')->daily()`. Use `Prunable` when deletion needs to trigger events; use `MassPrunable` for speed when events are not needed.

### 15. Incorrect Use of Eloquent `$with` Property

Scan models for the `$with` property that auto-eager-loads relationships on every query.

**Why it matters:** `protected $with = ['user', 'comments']` loads those relationships on every single query for that model, including queries where the relationships are not needed. This is hidden overhead that is easy to forget about because it does not appear in the query code.

**What to flag:**
- Models with `$with` containing relationships that are clearly not needed on every query
- Models with `$with` containing heavy relationships (has-many with many records, nested relationships)
- Repository evidence that `$with` is causing over-fetching across different contexts (API, admin, CLI)
- Do NOT flag `$with` on models that genuinely always need specific relationships

**Suggestion:** Remove `$with` and use explicit `->with()` in each query that needs the relationship when the relationship is not truly universal. If a model is received from another function and relationships need to be added later, use `->load()` on the existing collection. Reserve `$with` for models where the relationship is genuinely needed in nearly all use cases.

### 16. Unoptimized Seed and Bulk Insert Patterns

Scan seeders and import code for per-record `create()` calls in loops.

**Why it matters:** Using `Model::create()` inside a large loop runs individual INSERT queries over and over. This matters for seeders, CSV imports, and data migration scripts.

**What to flag:**
- `Model::create()` or `->save()` inside loops processing more than roughly 100 records
- Factory `::create()` in large loops in seeders
- CSV/Excel import code that processes rows one at a time with individual `create()` calls
- Do NOT flag small seeders or loops where model events must fire per record

**Suggestion:** Use `Model::insert()` with chunking for bulk operations: `$records->chunk(500)->each(fn($chunk) => Model::insert($chunk->toArray()))`. Note: `insert()` does not trigger model events, does not auto-set timestamps, and does not return model instances. Add timestamps manually if needed. For imports where validation matters per record, batch validated records and insert in chunks rather than one at a time.

## Evidence Expectations

Before making performance claims, prefer repository-visible evidence over assumptions:
- Check whether migrations suggest large tables, wide rows, JSON/text-heavy columns, or uniqueness constraints
- Look for tests, comments, metrics screenshots, Debugbar output, or existing optimization notes in the repo
- When reviewing a live codebase, use query inspection tools such as `toRawSql()`, `DB::listen()`, query logs, or `->explain()` when available before making strong claims about bottlenecks
- If the repository does not provide enough evidence, phrase the finding as a likely risk or optimization opportunity, not a proven production issue

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

- **High** — Clear performance bugs or correctness hazards: N+1 queries in production views, filtering or iterating huge datasets inefficiently, sensitive data exposed in API responses, multi-step writes that can leave inconsistent state, uniqueness-sensitive writes lacking database-backed guarantees
- **Medium** — Meaningful performance or maintainability issues: missing casts causing type bugs, hardcoded string constants for statuses/roles, `SELECT *` in API endpoints with large payloads, repeated query logic across 3+ files, mass assignment misconfigurations, accessors triggering hidden queries
- **Low** — Context-dependent improvements: missing pruning for disposable tables, suboptimal seed performance, `$with` over-fetching, minor relationship pattern improvements, modern Eloquent helpers that could simplify existing code

### What's Done Well

End with a short section acknowledging patterns the project already follows correctly. This prevents the audit from feeling like a list of complaints and validates good decisions.

## Important Guidelines

- Do NOT flag simple CRUD code. A controller method that does `Post::findOrFail($id)` and returns a view is fine.
- Do NOT suggest optimization for queries on small tables unless the pattern would cause issues at scale.
- Do NOT flag patterns where the suggested alternative does not exist for the project's Laravel version — check `composer.json`.
- Do NOT treat the absence of casts, scopes, or API Resources as a bug by itself. Flag them only when the missing feature causes a real problem.
- Do NOT treat older-but-valid syntax (`$casts`, `scopeXxx()`, etc.) as a problem by itself. Prefer modern syntax only when it matches the project's Laravel version and coding style.
- DO read enough of each file to understand context before flagging. A query with `->get()` followed by collection methods might be operating on a guaranteed-small dataset.
- DO check which Laravel version the project uses (`composer.json`) — features like `casts()` method, `#[Scope]` attribute, `withWhereHas()`, `preventLazyLoading()`, and `shouldBeStrict()` are version-dependent.
- DO consider the project size. A 5-model CRUD app does not need query caching, bulk insert optimization, or pruning strategies.
- DO check migration files to understand the database schema before flagging missing casts or incorrect column assumptions.
- DO check migrations for unique indexes before claiming `firstOrCreate()` / `updateOrCreate()` usage is unsafe.
- Present findings as suggestions, not mandates. Eloquent usage depends on project scale, team preferences, and performance requirements.
