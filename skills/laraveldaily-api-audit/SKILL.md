---
name: laraveldaily-api-audit
description: This skill should be used when the user asks to "audit Laravel API", "review API architecture", "check API code", "find API issues", "API review", or wants to analyze a Laravel API project for improvements. Scans the Laravel API codebase and reports high-signal findings with actionable suggestions.
---

# Laravel API Audit

Analyze a Laravel API project's code against proven API development practices. Scan the relevant PHP source files (excluding `vendor/`, `node_modules/`, `storage/`) and produce a structured report of findings with actionable suggestions.

This is NOT a security audit and NOT a general Laravel structure audit. Focus on API-specific behavior: contracts, JSON responses, validation, error handling, authentication, performance, testing, and route design.

## Before You Flag Anything

Read enough of the codebase to understand the project context first:

- Check `composer.json` and core bootstrapping to confirm the Laravel version and installed packages
- Detect how the API is mounted: `routes/api.php`, `routes/web.php`, custom route files, or routes loaded from `bootstrap/app.php`
- Detect the authentication approach before judging auth code: Sanctum, Passport, JWT, session/cookie auth, custom tokens, or no auth
- Check route groups and global middleware before flagging individual routes
- Check exception rendering and middleware before flagging controller-level JSON issues
- Distinguish public APIs, internal APIs, SPA backends, mobile backends, and webhook-only APIs

Present findings as suggestions, not mandates. API design involves trade-offs, and intentional consistency matters.

## What to Check

### 1. Missing API Resources or Unclear Response Transformation

Scan controllers for raw model responses or hand-built transformations that make the API contract hard to control.

**Why it matters:** Returning raw Eloquent models or manually transforming arrays across many endpoints makes the contract harder to reason about and easier to break accidentally. API Resources centralize transformation and help control exposed fields.

**What to flag:**
- `return $model`, `return Model::all()`, or similar raw model returns when that exposes unstable or internal fields
- `response()->json([...])` with repeated hand-built field mapping across multiple endpoints
- `->toArray()` with manual field picking or transformation in controllers
- The same model returned from multiple endpoints with inconsistent field selection
- Relationships accessed in Resources without `whenLoaded()` when that risks lazy loading or accidental exposure

**Suggestion:** Use API Resources when the response shape needs to be controlled, reused, or kept consistent. Define explicit fields in `toArray()`, use `ResourceClass::collection()`, and prefer `$this->whenLoaded()` for relationships. Separate Resources for index vs show endpoints can be reasonable.

**Exceptions:**
- A trivial endpoint like `['status' => 'ok']` does not need a Resource
- A small internal API with intentionally simple raw JSON is not automatically wrong
- Do not flag raw model returns if the model clearly hides sensitive fields and the project uses that pattern consistently

### 2. Incorrect or Inconsistent HTTP Status Codes

Scan controller return statements and exception handling for missing or semantically misleading HTTP status codes.

**Why it matters:** API consumers rely on status codes to understand outcomes without inspecting every response body. Incorrect codes make error handling brittle and reduce interoperability.

**What to flag:**
- Create endpoints returning 200 when the API clearly treats the operation as resource creation and returns the created entity
- Validation or business-rule failures returned as 200 with an error payload
- Server exceptions caught and returned as 200 with patterns like `'success' => false`
- Delete endpoints returning a success body with 200 when the rest of the API consistently uses 204, or vice versa
- Mutation endpoints with inconsistent status code conventions across similar controllers

**Suggestion:** Prefer semantically correct and consistent status codes:
- **200** for successful reads and many updates
- **201** for successful creation when a new resource is created
- **204** for successful delete or update with no body
- **422** for validation failures
- **404** for not found
- **401/403** for authentication/authorization failures
- **429** for throttling

**Exceptions:**
- `200` on update or delete is not automatically wrong if the endpoint intentionally returns a response body
- Judge consistency within the same API before suggesting changes

### 3. Inconsistent JSON Response Structure

Scan API controllers, Resources, and exception responses for inconsistent response shapes across the same API surface.

**Why it matters:** When similar endpoints return different JSON structures without a clear reason, consumers need custom parsing logic and the API becomes harder to use.

**What to flag:**
- Some endpoints return `{ "data": ... }` while similar endpoints return bare arrays or custom top-level keys for no clear reason
- Success responses mix incompatible conventions such as `"success": true`, bare arrays, and Resource responses across the same API
- Error responses vary arbitrarily between `{ "error": ... }`, `{ "message": ... }`, and custom shapes without a consistent pattern
- Pagination metadata is custom in some places and standard Laravel `links` / `meta` in others

**Suggestion:** Pick one response convention per API surface and apply it consistently. Laravel API Resources are a simple default, but intentional alternatives are valid too. For errors, prefer one predictable structure and let Laravel's exception handling do most of the work when possible.

**Exceptions:**
- Do not force a `data` wrapper if the project intentionally uses bare JSON or a spec like JSON:API
- Do not compare unrelated surfaces such as public API vs admin AJAX vs web responses

### 4. API Endpoints Returning HTML Error Pages

Scan API routes, middleware, and exception rendering for cases where API consumers may receive HTML instead of JSON.

**Why it matters:** API clients expect machine-readable errors. HTML error pages are difficult to parse and often indicate that API error handling is not wired consistently.

**What to flag:**
- API routes that can reach the default HTML exception renderer without JSON expectations being established
- Custom exception handling that renders HTML for routes that are clearly part of the API
- Not-found or authorization failures that leak framework/model internals in the response body
- Tests or code paths showing API endpoints require an `Accept: application/json` header to avoid HTML, with no project-level handling

**Suggestion:** First check whether the project already solves this globally through route groups, middleware, or exception rendering. If not, suggest a project-level solution: a middleware that ensures JSON expectations for API routes, exception rendering based on route patterns, or feature tests that assert JSON for API failures.

**Important:** Do not prescribe one exact fix. The goal is consistent JSON errors for API clients, not a specific middleware implementation.

### 5. N+1 Queries and Missing Eager Loading

Scan controllers and Resources for relationship access without matching eager loading.

**Why it matters:** Collection endpoints amplify N+1 problems quickly. APIs often serialize relationships, so missed eager loading can turn one request into dozens of queries.

**What to flag:**
- Queries using `all()`, `get()`, or `paginate()` without `with(...)` when the response accesses relationships
- Resources using `$this->relation->field` directly instead of safer conditional loading patterns
- Nested relationship access in Resources without matching eager loading in the caller
- Controllers eager loading some but not all relationships needed by the Resource

**Suggestion:** Pair Resource relationship access with controller-level eager loading. Use `with()` / `load()` in the query layer and `whenLoaded()` in Resources to avoid accidental lazy loading.

**Exceptions:**
- Do not flag single-record endpoints unless the relationship access is clearly expensive or repeated
- Do not flag relation access that is intentionally lazy and only used in a narrow non-collection path

### 6. Missing or Weak Validation on Write Endpoints

Scan POST, PUT, PATCH, and other write endpoints for missing validation or unsafe data handling.

**Why it matters:** API clients can send arbitrary payloads. Without validation, bad input can cause SQL exceptions, silent bad data, or unstable API behavior.

**What to flag:**
- `Model::create($request->all())`, `$model->update($request->all())`, or similar mass assignment from unvalidated input
- Write endpoints using request input directly with no validation layer
- Large inline validation blocks in controllers that would be clearer as Form Requests
- Repeated validation logic across multiple endpoints
- File uploads with weak or missing file validation

**Suggestion:** Use Form Requests for non-trivial validation, access clean data through `$request->validated()`, and keep validation consistent across similar endpoints.

**Exceptions:**
- Small inline validation in a simple internal endpoint can be acceptable
- Do not flag `$request->validate([...])` automatically; flag it when it becomes large, repeated, or mixed with other controller concerns

### 7. Unsafe Filtering, Sorting, and Query Parameters

Scan endpoints that accept query parameters for filtering, sorting, includes, pagination, or search.

**Why it matters:** APIs often expose dynamic query controls. Without validation or whitelisting, these endpoints become fragile, inconsistent, and prone to invalid SQL or accidental data exposure.

**What to flag:**
- `orderBy($request->sort, $request->direction)` without whitelisting allowed columns and directions
- Unvalidated `per_page`, `page`, `include`, `filter`, or `search` parameters used directly in queries
- Conditional query logic spread across controllers with no normalization or validation
- Arbitrary relation includes with no allowlist

**Suggestion:** Validate query parameters with a Form Request or explicit validation, whitelist sortable/filterable fields, cap `per_page`, and centralize query shaping when it becomes complex.

### 8. Route Model Binding Not Used Where It Would Simplify the API

Scan API controllers for manual lookups from route parameters when implicit binding would make the code clearer and more consistent.

**Why it matters:** Route model binding reduces boilerplate and standardizes not-found behavior, but it is not appropriate for every query.

**What to flag:**
- `Model::find($id)`, `findOrFail($id)`, or equivalent boilerplate where the route parameter maps directly to a model
- Manual `404` JSON responses for simple primary-key lookups
- Inconsistent use of route model binding across otherwise similar endpoints

**Suggestion:** Type-hint the model in the controller signature when the lookup is straightforward. Use custom route keys for slugs where appropriate.

**Exceptions:**
- Do not flag custom queries that intentionally enforce tenant scoping, ownership, published-only visibility, or nested constraints
- Do not force binding when the controller needs query customization beyond a simple route lookup

### 9. Missing Transactions on Multi-Step Writes

Scan write endpoints and action/service classes for multi-step mutations without transactional boundaries.

**Why it matters:** Many API endpoints create or update several records, attach relations, move files, or dispatch side effects. Without a transaction, partial writes leave the system inconsistent when one step fails.

**What to flag:**
- Create/update flows that write multiple models without `DB::transaction()`
- Endpoints that create a record and then sync/attach related records with no transactional protection
- Balance/inventory/status changes spread across several writes
- Deleting records plus related cleanup where a failure could leave orphaned state

**Suggestion:** Wrap tightly related database writes in `DB::transaction()` when partial success would be a bug. Keep external side effects in mind and dispatch them after commit when appropriate.

**Exceptions:**
- Do not suggest transactions for single-write CRUD with no dependent writes
- Do not force transactions around flows dominated by external HTTP calls where compensating actions are the real concern

### 10. Rate Limiting Gaps on Abuse-Prone Endpoints

Scan route files and middleware configuration for rate limiting where abuse is likely to matter.

**Why it matters:** Public APIs, auth endpoints, and expensive endpoints are vulnerable to abuse. Lack of throttling on these routes can lead to unnecessary load and poor resilience.

**What to flag:**
- Public API groups with no throttling at all
- Login, register, password reset, OTP, or token issuance endpoints without stricter throttling
- Expensive search/export/reporting endpoints with no protection
- Projects with route-level throttling needs but only a generic default applied everywhere

**Suggestion:** Check the route groups first. If throttling is missing where it matters, suggest `throttle` middleware or dedicated `RateLimiter::for()` definitions with limits based on route type and user/IP.

**Exceptions:**
- Do not flag every internal API or low-risk backend endpoint for lacking custom throttle rules
- Do not require custom `RateLimiter::for()` definitions if the project already has sensible inherited defaults

### 11. Authentication Guard and Token Handling Issues

Detect the API's auth approach first, then scan for mistakes within that chosen approach.

**Why it matters:** Authentication bugs often appear as intermittent 401s, broken sessions, or overexposed login responses. The right checks depend on the actual auth stack in use.

**What to flag:**
- Sanctum projects missing `HasApiTokens`, missing `auth:sanctum` protection where needed, or exposing too much data in login/register responses
- Password verification that compares raw strings instead of using `Hash::check()`
- Protected endpoints missing the expected auth middleware for the project's chosen guard
- Cookie-based SPA auth that appears miswired: missing `statefulApi()`, missing stateful domains, or CORS/credentials settings that contradict the auth strategy
- Token issuance endpoints returning full user payloads with fields that should not be exposed

**Suggestion:** First identify whether the project uses Sanctum, Passport, JWT, session auth, or custom tokens. Then judge the implementation against that approach. Recommend minimal login/register responses, correct guard middleware, and clear separation between public and protected routes.

**Important:** Do not assume Sanctum unless the codebase actually uses it.

### 12. API Route Design and Conventions

Scan route files for conventions that hurt clarity or consistency in the API surface.

**Why it matters:** Predictable route design helps API consumers and maintainers. Laravel's API helpers are useful, but not every endpoint should be forced into CRUD resource patterns.

**What to flag:**
- Standard CRUD controllers defined with repetitive manual routes instead of `Route::apiResource()` when that would clearly reduce noise
- `Route::resource()` used for API-only controllers, adding unused `create` and `edit` routes
- Inconsistent naming and URL patterns across similar endpoints
- Clearly awkward RPC-style endpoints where a more conventional route would improve clarity
- Missing versioning only when the project clearly has multiple external consumers or backward-compatibility pressure

**Suggestion:** Use `Route::apiResource()` for conventional CRUD where it fits. Keep naming and URL patterns consistent. Use explicit custom endpoints for actions like imports, exports, bulk operations, webhooks, and callbacks when those are the more honest design.

**Exceptions:**
- Do not auto-flag webhook, callback, search, export, bulk-action, or non-CRUD workflow endpoints for not being REST-pure
- Do not force versioning on small internal APIs

### 13. CORS Misconfiguration

Scan CORS configuration only if the API is consumed cross-origin or uses browser-based clients.

**Why it matters:** CORS settings can either block legitimate frontend clients or conflict with cookie-based authentication if configured incorrectly.

**What to flag:**
- Cookie/session-based APIs using wildcard origins
- Missing auth-related paths in CORS config when browser clients need them
- `supports_credentials` mismatched with a cookie-based SPA setup
- Browser-facing APIs with no clear CORS configuration when one is needed

**Suggestion:** Align CORS settings with the actual client type. For cookie-based SPAs, use explicit origins and credentials support. For token-only public APIs, broader origin settings may be acceptable.

**Exceptions:**
- Do not flag missing CORS config in a server-to-server API with no browser consumers

### 14. Missing API Documentation Strategy

Scan for whether the project has a documentation approach appropriate to its consumers.

**Why it matters:** Public or cross-team APIs benefit from docs, examples, and an inspectable contract. Internal-only APIs may not need a full documentation package.

**What to flag:**
- Public or partner-facing APIs with no clear documentation strategy
- Documentation tooling installed but obviously not maintained or not connected to the actual routes
- Endpoints with complex request/response shapes and no examples anywhere
- Error responses undocumented when external consumers rely on them

**Suggestion:** If the API is public, partner-facing, or shared across teams, suggest an appropriate documentation approach. Scramble and Scribe are practical Laravel-first choices; OpenAPI-driven tools are appropriate when the team wants a formal spec.

**Exceptions:**
- Do not flag every small internal API for lacking Swagger or OpenAPI packages

### 15. Missing Pagination on Unbounded Collection Endpoints

Scan collection endpoints for unbounded result sets.

**Why it matters:** Returning all rows from a growing dataset can turn into memory, latency, and usability problems as the application grows.

**What to flag:**
- Index endpoints using `all()` or `get()` for clearly growing datasets
- Custom collection endpoints returning large unbounded result sets
- `per_page` controls with no maximum cap
- Large default page sizes with no clear justification

**Suggestion:** Use `paginate()` or `cursorPaginate()` for growing datasets, validate `per_page`, and apply a sensible maximum.

**Exceptions:**
- Do not flag small bounded lookup data such as countries, roles, status lists, or similar reference tables
- Do not assume every collection must be paginated; judge based on expected growth and consumer needs

### 16. Missing or Weak API Test Coverage

Scan the test suite for HTTP/API feature coverage.

**Why it matters:** API contracts are easiest to break silently. Feature tests are often the strongest defense against regressions in JSON shape, auth behavior, validation, and status codes.

**What to flag:**
- API endpoints with little or no feature test coverage
- Tests covering only happy paths but not auth failures, validation failures, not-found cases, or authorization failures
- No assertions around JSON shape, status codes, or pagination metadata for important endpoints
- Critical auth or mutation flows with no automated tests

**Suggestion:** Add feature tests for the highest-risk endpoints first. Prioritize validation failures, unauthorized access, not-found responses, JSON structure, and create/update/delete flows.

### 17. `env()` Used Outside Config Files

Scan all PHP files outside `config/` for direct `env()` calls.

**Why it matters:** This is a production bug, not a style preference. After `php artisan config:cache`, `env()` is unreliable outside config files.

**What to flag:**
- Any `env('...')` call outside `config/*.php`
- Especially risky API-related cases: auth domains, CORS origins, API keys, and callback/base URLs

**Suggestion:** Move the value into a config file and read it through `config(...)` everywhere else.

## Output Format

Present findings in this structure:

### Summary

A brief overview: total findings count, top 2-3 most impactful areas to address, and the project context you detected (Laravel version, auth approach, public/internal API, and route setup).

### Findings by Category

For each category that has findings:

**Category Name**

| Severity | File | Line(s) | Issue | Suggestion |
|----------|------|---------|-------|------------|
| High/Medium/Low | path/to/file.php | 42-58 | What was found | What to do |

### Severity Levels

- **High** — Clear production bugs or contract hazards: HTML error pages from API endpoints, unsafe unvalidated write input, missing auth on sensitive routes, broken auth guard setup, unbounded multi-step writes without transactions, `env()` outside config
- **Medium** — Meaningful API quality or maintainability issues: inconsistent response structure, N+1 queries on important endpoints, missing pagination on growing datasets, abuse-prone routes without throttling, missing tests around important API behavior
- **Low** — Context-dependent improvements: route model binding opportunities, Resource extraction for consistency, documentation strategy improvements, route convention cleanup

### What's Done Well

End with a short section acknowledging patterns the project already follows correctly. A good audit should confirm sound decisions, not only list problems.

## Important Guidelines

- Do NOT nitpick small internal APIs. A 3-endpoint private API does not need versioning, OpenAPI tooling, and custom response helpers.
- Do NOT flag simple endpoints just because a more abstract Laravel pattern exists.
- Do NOT assume Sanctum, API Resources, or `api.php` are required. Verify the project's actual approach first.
- Do NOT flag conventions as bugs. Separate production risks from style or consistency suggestions.
- DO read enough surrounding code before flagging a file. Global middleware, exception handlers, and route groups often explain local code.
- DO prefer high-signal findings. Ten accurate findings are better than forty generic suggestions.
