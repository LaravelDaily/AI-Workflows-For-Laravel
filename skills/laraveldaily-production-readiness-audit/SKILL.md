---
name: laraveldaily-production-readiness-audit
description: Perform a comprehensive security audit of all project files in the current working directory.
---

# Skill: laraveldaily-production-readiness-audit

## Purpose

Audit a Laravel codebase for production-readiness risks using **only repository contents**.

This skill MUST:
- Run locally
- NOT assume anything about production environment
- NOT rely on `.env` values
- ONLY report high-confidence issues detectable from code

---

## Core Principle

> Only report issues that can be inferred from the repository with high confidence.  
> Do NOT assume server configuration, deployment process, or runtime environment.

---

## Output Format

Return results in this structure:

## ❌ Critical Findings
- ...

## ⚠️ Warnings
- ...

## ℹ️ Notes
- ...

---

## Severity Rules

### ❌ Critical
High confidence production risk or data leak.

### ⚠️ Warning
Strong code smell or likely issue.

### ℹ️ Note
Non-blocking observation.

---

## Audit Checks

---

## 1. Secrets & Sensitive Data (CRITICAL)

Search entire repo for:

### Patterns:
- `.env` files committed (NOT `.env.example` — that is expected)
- API keys / tokens:
  - `sk_live_`
  - `AKIA`
  - `-----BEGIN PRIVATE KEY-----`
- Hardcoded credentials in code/config (literal strings, NOT `env()` references)
- Logs committed:
  - `storage/logs/*.log`

### Exclusions (DO NOT flag):
- `.env.example` — this is a template, not a secret
- `env('SOME_SECRET')` or `env('API_TOKEN')` inside `config/*.php` — these are config placeholders referencing environment variables, not leaked secrets
- Constants or variable names containing SECRET/TOKEN (e.g., `$tokenSecret`, `TOKEN_LIFETIME`, `CSRF_TOKEN`) — flag only actual secret values, not names

### Output:
- ❌ if any real secret values detected (actual keys, passwords, tokens in plaintext)
- ⚠️ if suspicious but ambiguous strings

---

## 2. Debug Code in Runtime (CRITICAL)

Search in:
- `app/`
- `routes/`
- `database/`

### Detect (function calls only — match `functionName(` as a standalone call):
- `dd(`
- `ray(`
- `var_dump(`
- `print_r(`
- `phpinfo(`

### Detect with care (high false-positive risk):
- `dump(` — ONLY flag if it is a standalone `dump(...)` call. Do NOT flag: `Dumpable` trait, `shouldDump()`, `$dumper`, or any method/variable name containing "dump".
- `exit(` — ONLY flag in controllers, middleware, jobs, and service classes. Do NOT flag in Artisan commands where `exit(0)` / `exit(1)` is a legitimate process exit pattern.
- `die(` — ONLY flag in controllers, middleware, jobs, and service classes.

### Rules:
- ❌ if inside controllers, middleware, jobs, service classes
- Ignore if inside `tests/` directory
- ⚠️ for `exit(` in Artisan commands (legitimate but worth noting)

---

## 3. env() Misuse (CRITICAL)

### Rule:
Search for `env(` usage outside allowed locations.

Allowed:
- `config/*.php`
- `bootstrap/app.php`
- `bootstrap/providers.php`

Flag:
- anywhere else (controllers, models, services, routes, views, etc.)

### Output:
- ❌ always (Laravel best-practice violation — after config caching, `env()` returns null outside config files)

---

## 4. Dangerous Seed Data (CRITICAL / WARNING)

Search:
- `database/seeders`
- `database/factories`

### Differentiate between factories and seeders:

**Factories** (expected for testing — lower severity):
- `Hash::make('password')` or `bcrypt('password')` in a factory is the Laravel default and is expected. Do NOT flag as critical.
- ⚠️ only if the factory uses real-looking credentials (actual email domains, non-placeholder passwords)

**Seeders** (may run in production — higher severity):
- ❌ if a seeder creates users with hardcoded passwords (`'password'`, `'123456'`, `'admin'`, `'secret'`)
- ❌ if a seeder assigns admin/superadmin roles with hardcoded credentials
- ⚠️ if a seeder uses hardcoded demo emails (`admin@example.com`, `test@test.com`) without environment guards

### Output:
- ❌ if clearly usable credentials in seeders
- ⚠️ if suspicious/demo data in seeders or non-default credentials in factories

---

## 5. Dependency Integrity (WARNING)

### Check:
- `composer.lock` exists
- frontend lockfile exists:
  - `package-lock.json`
  - `yarn.lock`
  - `pnpm-lock.yaml`

### Detect:
- dev packages used in runtime code

### Output:
- ⚠️ missing lockfiles
- ⚠️ runtime usage of dev-only packages

---

## 6. Asset Build Consistency (WARNING)

### Detect mismatches:

- Blade uses:
  - `@vite(...)` but no `vite.config.*`
  - `mix(...)` but no Mix config

- `package.json` exists but:
  - no `build` script

- references to missing assets

### Output:
- ❌ if clearly broken references
- ⚠️ if incomplete setup

---

## 7. Production-Breaking Code Patterns (WARNING)

Search for:

### Warning:
- `abort(500)` — sometimes intentional but worth reviewing
- comments containing:
  - `TODO remove` or `TODO: remove`
  - `FIXME`
- Do NOT flag: `temporary` as a standalone word (matches legitimate uses like `temporaryUrl()`, `temporarySignedRoute()`). Only flag comments explicitly saying "this is temporary" or "temporary hack/fix/workaround".
- Do NOT flag: `debug` as a standalone word (matches `debugbar`, `debug_backtrace`, log channel names). Only flag comments explicitly saying "debug code" or "remove debug".

### Note:
- This check does NOT flag `dd(`, `die(`, `dump(`, or `exit(` — those are covered in Check 2 to avoid duplicate findings.

---

## 8. Queue & Job Smells (WARNING)

### Detect:
- Jobs using closures (non-serializable)
- Jobs doing heavy sync work
- Missing retry/backoff/timeout hints

### Output:
- ⚠️ only (no assumptions about infra)

---

## 9. Logging & Exception Handling (WARNING)

### Detect:
- empty `catch` blocks
- swallowed exceptions
- broad `catch (\Exception)` with no logging
- logs containing sensitive data patterns

### Output:
- ⚠️

---

## 10. Missing Rate Limiting on Auth Routes (WARNING)

### Detect:
- Login, registration, and password reset routes without `throttle` middleware
- Check `routes/web.php`, `routes/auth.php`, and Fortify configuration
- If Fortify is used, check if `limiter()` is configured in `FortifyServiceProvider`

### Output:
- ⚠️ if auth routes lack throttle/rate limiting middleware

---

## 11. Sensitive Data Exposure in Models (WARNING)

### Detect:
- Models without `$hidden` property that have sensitive columns (check migrations or schema for: `password`, `remember_token`, `two_factor_secret`, `two_factor_recovery_codes`, `secret`, `api_token`)
- Models that are returned directly in API responses (via routes or resources) without ensuring sensitive fields are hidden
- Eloquent API Resources that include sensitive fields

### Output:
- ⚠️ if sensitive columns exist on a model but `$hidden` does not cover them

---

## Execution Strategy

1. Scan file tree
2. Grep for patterns
3. Parse:
   - composer.json
   - package.json
   - routes
   - app code
4. Classify findings by severity
5. Deduplicate results (do NOT report the same issue under multiple checks)
6. Output structured report

---

## Explicit Non-Goals

DO NOT check:

- `.env` values correctness
- APP_ENV / APP_DEBUG values
- cache/session/queue drivers
- Redis / DB availability
- file permissions
- server config (Nginx, Apache)
- SSL / HTTPS
- cron jobs
- deployment scripts
- whether `config:cache` is run

---

## Final Rule

If unsure → DO NOT report.

Prefer missing an issue over producing a false positive.
