---
name: laraveldaily-permissions-audit
description: This skill should be used when the user asks to "audit Laravel permissions", "review roles and permissions", "check authorization code", "find permission issues", "permissions review", "audit access control", or wants to analyze a Laravel project's roles/permissions implementation for improvements. Scans all PHP source files and reports findings with actionable suggestions.
---

# Laravel Roles & Permissions Audit

Analyze a Laravel project's roles, permissions, and authorization implementation. Scan all PHP source files (excluding `vendor/`, `node_modules/`, `storage/`) and produce a structured report of findings with actionable suggestions.

This is NOT a security penetration test. Focus on authorization architecture, role/permission patterns, and access control implementation quality.

## What to Check

### 1. UI-Only Authorization Without Backend Protection

Scan for cases where authorization is checked in Blade templates but NOT enforced in the corresponding controller methods.

**Why it matters:** Hiding a button with `@can` or `@if(auth()->user()->isAdmin())` does not prevent a direct HTTP request to the endpoint. Every authorization check in the UI must have a matching backend check. This is a real security gap, not a style issue.

**What to flag:**
- `@can`, `@if(auth()->user()->...)`, or role/permission checks in Blade templates where the corresponding controller method has no `Gate::authorize()`, `$this->authorize()`, `#[Authorize]`, Policy check, or `can` middleware on the route
- Form actions or links hidden with `@can` in Blade but the POST/PUT/DELETE endpoint has no authorization
- Admin-only navigation items hidden with `@if` checks but the underlying routes have no middleware or Policy protection

**Suggestion:** Every `@can` check in Blade should have a matching authorization check on the backend. Use `Gate::authorize('ability', $model)` in the controller, `#[Authorize]` attributes on controller methods, or `->can('ability,model')` on the route definition. The UI check is cosmetic — the backend check is the actual security.

### 2. Inline Role/Permission Checks Instead of Policies

Scan controllers and middleware for manual role and permission checks instead of using Laravel's Policy system.

**Why it matters:** Authorization logic scattered across controllers is easy to forget in new endpoints, hard to audit, and cannot be reused in Blade (`@can`), route definitions, or Gates. Policies centralize authorization per model and are auto-discovered in Laravel 11+.

**What to flag:**
- `if ($user->role === 'admin')`, `if ($user->is_admin)`, `if ($user->hasRole(...))` directly in controller methods to decide access
- `abort(403)` or `abort_if`/`abort_unless` with role or ownership conditions in controllers
- `if ($user->id !== $model->user_id)` ownership checks in controllers
- Same authorization condition copy-pasted across multiple controller methods
- Ignore middleware-level role checks (e.g., `role:admin` middleware on route groups) — those are appropriate for broad route protection

**Suggestion:** Create a Policy class using `php artisan make:policy PostPolicy --model=Post`. Move access decisions to Policy methods (`viewAny`, `view`, `create`, `update`, `delete`). Call them via `Gate::authorize()`, `#[Authorize]` attributes, `@can` in Blade, or `->can()` on route definitions. In Laravel 11+, Policies are auto-discovered — no manual registration needed.

### 3. Checking Roles Instead of Permissions

Scan for authorization code that checks role names rather than permission names.

**Why it matters:** Checking `hasRole('admin')` ties your logic to a specific role. If you later add a "manager" role that should also access that feature, you must find and update every `hasRole('admin')` check. Checking `hasPermissionTo('edit-post')` instead means you only update the role-permission mapping in one place (the seeder or admin UI). This is the fundamental principle of permission-based access control.

**What to flag:**
- `$user->hasRole('...')` or `$user->hasAnyRole([...])` used to gate specific actions (not broad area access)
- Policy methods that check role names: `$user->role === 'admin'` or `$user->hasRole('admin')` instead of checking a permission
- Middleware using `role:admin` on individual routes rather than route groups for broad sections
- Multiple role checks combined with `||`: `$user->hasRole('admin') || $user->hasRole('manager')` — this is a sign the check should be a permission

**Exception:** Checking roles is acceptable for broad area access (e.g., `role:admin` middleware on an entire admin route group) or when the role itself IS the distinction (master admin bypass). Do not flag `Gate::before()` super-admin bypasses or route-group-level role middleware.

**Suggestion:** Define granular permissions (`create-post`, `edit-post`, `delete-post`) and assign them to roles in a seeder or admin UI. Check `$user->hasPermissionTo('edit-post')` or `$user->can('edit-post')` in application logic. This way, adding a new role that can edit posts requires only a seeder change, not a code change.

### 4. Boolean `is_admin` Column When Multiple Roles Exist

Scan for boolean admin flags on the User model when the application clearly has more than two user types.

**Why it matters:** A boolean `is_admin` column works for a strict two-role app (admin vs user). But when the application has admin, manager, editor, or any third role, a boolean cannot represent it. This becomes a source of hacks like adding `is_manager`, `is_editor` columns.

**What to flag:**
- `is_admin` boolean column on users table alongside other role-like columns (`is_manager`, `is_editor`, `is_moderator`)
- Multiple boolean role columns on the same table
- Code that checks `$user->is_admin` alongside other role-like conditions, suggesting more than two user types exist
- `is_admin` boolean used in a project that clearly has 3+ distinct user types based on its controllers, routes, or middleware

**Do NOT flag:** A simple app with genuinely only two user types (admin and regular user) using `is_admin` — this is a valid and simple approach for small projects.

**Suggestion:** Replace boolean columns with a proper roles system. For a fixed set of roles: add a `role` column backed by a PHP Enum and a `roles` DB table. For flexible/growing roles: use Spatie `laravel-permission` package. Choose one-to-many (user has one role) or many-to-many (user has multiple roles) based on whether users genuinely need to hold multiple roles simultaneously.

### 5. Hardcoded Role/Permission Strings Without Enums

Scan for role and permission names used as raw strings scattered across the codebase.

**Why it matters:** When role names are strings like `'admin'` or `'clinic-owner'` scattered across Policies, middleware, seeders, and Blade templates, a typo compiles silently and fails at runtime. PHP Enums provide type safety, IDE auto-completion, and a single source of truth for all valid roles and permissions.

**What to flag:**
- `hasRole('admin')`, `hasPermissionTo('edit-post')`, `assignRole('user')` with raw strings instead of Enum references
- Spatie's `Role::create(['name' => 'admin'])` in seeders using raw strings instead of Enum values
- `@can('edit-post')` in Blade or `Gate::define('edit-post', ...)` with hardcoded strings
- `in:admin,editor,user` validation rules with hardcoded role lists
- Same role/permission string appearing in 3+ different files

**Suggestion:** Create `App\Enums\Role` and `App\Enums\Permission` as string-backed PHP Enums. Use `Role::ADMIN->value` in seeders, `Permission::EDIT_POST->value` in Policies, and reference the Enum class in validation rules. When using Spatie, use `enum_value()` or `->value` when passing to Spatie methods that expect strings.

### 6. Missing Data Scoping (Authorization Without Query Filtering)

Scan for projects that check permissions but do not scope database queries to the authorized data.

**Why it matters:** A Policy that says "this user can view tasks" does not prevent them from viewing ALL tasks. If users should only see their own data (or their team's data), you need Global Scopes or query-level filtering in addition to Policies. Without scoping, a user who guesses another record's ID can access it even with Policies in place (Policies check ability, not record visibility by default).

**What to flag:**
- Controllers that query `Model::all()` or `Model::paginate()` without any user/team scoping when different users should see different records
- `Model::findOrFail($id)` without a scope limiting to the user's own records or team
- Multi-tenant or multi-team apps where queries don't filter by `team_id`, `organization_id`, or `user_id`
- Policies that authorize the action but the controller still fetches unscoped data

**Suggestion:** Use Global Scopes to automatically filter queries by the authenticated user's team or ownership. For example, a `TeamScope` that adds `->where('team_id', auth()->user()->current_team_id)` to all queries on team-scoped models. This provides defense in depth: the Policy authorizes the action, and the scope ensures only permitted records are visible. Note: Global Scopes cause 404 (record not found) instead of 403 (forbidden) for cross-team/cross-user access — this is usually desirable as it doesn't leak that the record exists.

### 7. Missing Authorization Middleware on Routes

Scan route files for unprotected routes that should require authorization.

**Why it matters:** Every route that serves authenticated content should be behind `auth` middleware at minimum. Routes that serve role-specific content should additionally have role or permission middleware, or the controller should use Policies. Unprotected routes are the most basic authorization gap.

**What to flag:**
- Routes inside `web.php` or `api.php` that point to controllers containing authorization checks but the routes themselves are not inside an `auth` middleware group
- Admin-prefixed routes (`/admin/*`) without `auth` middleware or role/permission middleware
- Resource routes where some methods need authorization but no middleware or Policy is applied
- Routes that rely solely on controller-level auth checks with no middleware — while this works, combining middleware for broad access (auth, role) with Policies for fine-grained access is more robust

**Suggestion:** Wrap authenticated routes in `Route::middleware('auth')` groups. For role-based areas, add role middleware on route groups: `Route::middleware(['auth', 'role:admin'])`. For fine-grained access, use Policies in controllers and `->can()` on individual route definitions. The layered approach (middleware for broad access + Policies for specific actions) is the most secure.

### 8. Overly Complex Role System for Simple Needs

Scan for over-engineered role/permission setups in small applications.

**Why it matters:** A 5-controller CRUD app with Spatie `laravel-permission`, 15 granular permissions, a role-permission seeder, and team-based scoping is over-engineered if the app only has "admin" and "user" roles. The right solution depends on the project's actual complexity. A simple `is_admin` column or a `role` Enum column is often sufficient.

**What to flag:**
- Spatie `laravel-permission` installed but only 2 roles and no permission-level checks anywhere (everything checks roles)
- Permission tables with entries that map 1:1 to roles (every permission assigned to exactly one role) — this means permissions add no value over just checking roles
- Complex role/permission seeders for an app with fewer than 5 controllers
- Team-based permissions (`teams` feature enabled in Spatie config) when the app has no multi-tenancy concept
- Repository pattern + Service layer + Action classes + DTOs all wrapping simple role checks

**Do NOT flag:** Consistent use of a permissions package in a growing application, even if current complexity is modest — planning ahead is reasonable if the codebase shows signs of growth.

**Suggestion:** Match the authorization approach to the project's actual complexity:
- **2 roles (admin/user):** `is_admin` boolean + simple middleware is fine
- **3-5 fixed roles:** `role` Enum column + Policies, no package needed
- **Multiple roles per user OR dynamic roles:** Spatie `laravel-permission`
- **Multi-tenant with per-team roles:** Spatie with teams feature enabled

### 9. Multiple Authenticatable Models Instead of Roles

Scan for multiple models extending `Authenticatable` (separate `Admin`, `Manager`, `Staff` models alongside `User`).

**Why it matters:** Having `Admin.php` and `User.php` as separate Authenticatable models means separate authentication guards, separate sessions, separate password resets, and every auth-related feature must be duplicated. In almost all cases, a single `User` model with a role column (and Policies for authorization) is simpler and more maintainable.

**What to flag:**
- Multiple models extending `Authenticatable` or `User` in the `app/Models/` directory
- Multiple auth guards in `config/auth.php` for different "user types" that are really roles
- Separate login forms, registration, or password reset for different user types that share the same database

**Exception:** Truly separate authentication domains (e.g., customers and internal admins on different databases, or API consumers vs web users) may justify separate models. Multi-tenancy setups with isolated databases are also a valid exception.

**Suggestion:** Consolidate to a single `User` model with a `role` column (PHP Enum). Use Policies and middleware for authorization. Use a single authentication guard. If using Spatie, use `assignRole()` on the single User model.

### 10. Spatie Teams Middleware Not Prioritized Correctly

Scan for projects using Spatie `laravel-permission` with teams enabled but missing the middleware priority configuration.

**Why it matters:** When Spatie's teams feature is enabled, the middleware that sets `setPermissionsTeamId()` must run BEFORE `SubstituteBindings`. If it runs after, route model binding resolves models before the team context is set, causing permission checks to fail or return wrong results. This is a silent production bug.

**What to flag:**
- `teams` set to `true` in `config/permission.php` without a corresponding `addToMiddlewarePriorityBefore(SubstituteBindings::class, TeamsPermissionMiddleware::class)` in `AppServiceProvider` (or equivalent middleware priority configuration)
- A custom teams permission middleware that calls `setPermissionsTeamId()` registered AFTER `SubstituteBindings` in the middleware stack
- Team-scoped permission checks that work in tests but may fail in HTTP requests due to middleware ordering

**Suggestion:** In `AppServiceProvider::boot()`, add:
```php
app(\Illuminate\Routing\Router::class)
    ->addToMiddlewarePriorityBefore(
        \Illuminate\Routing\Middleware\SubstituteBindings::class,
        \App\Http\Middleware\TeamsPermissionMiddleware::class
    );
```
This ensures the team context is set before route model binding resolves any models.

### 11. Missing Authorization Tests

Scan test files for adequate coverage of authorization rules.

**Why it matters:** Authorization is one of the most critical parts of an application but is often untested. Without tests, a refactor can silently remove a permission check, or a new endpoint can be deployed without any access control. Tests should verify both "allowed" and "forbidden" paths.

**What to flag:**
- Controller test files that test CRUD operations but never assert 403 (Forbidden) responses
- No tests for role-based access at all (search for `assertForbidden`, `assertStatus(403)`, `assertUnauthorized`, `assertStatus(401)`)
- Tests that always authenticate as admin — never testing that restricted users are actually restricted
- Policy classes that exist but have no corresponding test
- Multi-team/multi-tenant apps with no tests verifying cross-team data isolation (user A cannot see user B's team data)

**Suggestion:** For each Policy or Gate, write tests covering:
1. **Allowed:** User with the right role/permission CAN perform the action
2. **Forbidden:** User without the right role/permission gets 403
3. **Unauthenticated:** Guest gets 401/redirect
4. **Scope boundaries:** User cannot access another user's/team's records (expect 404 if using Global Scopes, 403 if using Policies alone)

Use Pest datasets to test multiple roles efficiently:
```php
dataset('forbidden_roles', ['user', 'viewer', 'patient']);
it('forbids non-admin roles', function (string $role) {
    $user = User::factory()->{$role}()->create();
    actingAs($user)->get('/admin/users')->assertForbidden();
})->with('forbidden_roles');
```

### 12. Registration/Onboarding Missing Default Role Assignment

Scan user registration logic for missing role assignment.

**Why it matters:** If new users are not assigned a default role during registration, they exist in a "no role" state. Depending on the authorization setup, this can either lock them out of everything (confusing) or — worse — bypass role checks entirely if the code only checks "does the user have role X" and falls through to allow access when no role exists.

**What to flag:**
- User registration actions (`CreateNewUser`, `RegisteredUserController`, or custom registration) that create a user without assigning any role
- Projects using Spatie `laravel-permission` where registration does not call `assignRole()`
- Projects with a `role_id` column where registration does not set it (and the column has no database default)
- Registration flow in a multi-team app that does not set the user's `current_team_id` or team association

**Suggestion:** Always assign a default role during registration. With Spatie: call `$user->assignRole(Role::USER)` immediately after creating the user. With a `role_id` column: set a `default()` in the migration AND set it explicitly in the registration action. In multi-team apps, also associate the user with their team and set `current_team_id`. Wrap all of this in a DB transaction.

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

- **High** — Security-impacting authorization gaps: missing backend protection for UI-hidden actions, unprotected routes serving sensitive data, no query scoping in multi-tenant apps, registration creating users with no role in a role-required system, Spatie teams middleware ordering bugs
- **Medium** — Architectural issues that hurt maintainability and correctness: checking roles instead of permissions, inline authorization scattered across controllers without Policies, hardcoded role strings without Enums, no authorization tests, missing data scoping for user-owned records
- **Low** — Context-dependent improvements: `is_admin` boolean in a project that might outgrow it, slight over-engineering of the role system, missing default role assignment when the column has a DB default anyway, minor opportunities to consolidate duplicate authorization checks

### What's Done Well

End with a short section acknowledging authorization patterns the project already follows correctly. This prevents the audit from feeling like a list of complaints and validates good decisions.

## Important Guidelines

- Do NOT nitpick simple apps. A 3-controller app with `is_admin` boolean is fine if it only has admin and user roles.
- Do NOT push Spatie on every project. Many apps work perfectly with Enums + Policies and no package.
- Do NOT flag every `hasRole()` call. Role checks are appropriate for broad area access (admin section middleware). Flag them only when checking roles for specific action authorization where permissions would be more flexible.
- DO check `composer.json` for the Laravel version — `#[Authorize]` attributes and Policy auto-discovery are Laravel 11+ features.
- DO check if `spatie/laravel-permission` is installed before making Spatie-specific suggestions.
- DO read enough of each file to understand context before flagging. A `hasRole('admin')` in a `Gate::before()` callback is a legitimate super-admin bypass, not a problem.
- DO consider the project's complexity level. Match your suggestions to what the project actually needs, not what a large enterprise app would need.
- Present findings as suggestions, not mandates. Authorization architecture has trade-offs — the audit surfaces opportunities, not requirements.
