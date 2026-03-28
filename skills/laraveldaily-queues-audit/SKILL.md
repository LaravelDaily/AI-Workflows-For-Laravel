---
name: laraveldaily-queues-audit
description: This skill should be used when the user asks to "audit Laravel queues", "review queue implementation", "check job code", "find queue issues", "queue review", or wants to analyze a Laravel project's queue/job usage for improvements. Scans all PHP source files and reports findings with actionable suggestions.
---

# Laravel Queues Audit

Analyze a Laravel project's queue and job implementation against proven patterns. Scan relevant PHP source files plus queue-related config, tests, env examples, and versioned deployment files (excluding `vendor/`, `node_modules/`, `storage/`) and produce a structured report of findings with actionable suggestions.

This is NOT a performance benchmark or full infrastructure audit. Focus on job code quality, queue patterns, failure handling, queueable classes, and correct use of Laravel's queue features. Only make deployment or operations findings when they are verifiable from files present in the repository.

## What to Check

### 1. Long-Running Operations Not Using Queues

Scan controllers, Livewire components, and route closures for synchronous operations that would benefit from being queued.

**Why it matters:** Long-running operations block the HTTP response, causing timeouts, poor UX, and wasted server resources. Queues process these operations in the background while the user gets an immediate response.

**What to flag:**
- Sending emails or notifications synchronously in controllers (e.g., `Mail::to(...)->send(...)`, `Notification::send(...)` where the Notification/Mailable class does NOT implement `ShouldQueue`)
- External API calls (HTTP client, SDK calls) inside controller methods that could fail or be slow
- File processing, PDF generation, CSV imports/exports, or image manipulation in controller methods
- Loops that perform I/O operations (sending multiple emails, making multiple API calls) inside a request lifecycle
- Ignore simple, fast operations — only flag operations likely to take more than a second or that are prone to external failure

**Suggestion:** Extract the operation into a Job class using `php artisan make:job`. Dispatch the job from the controller and return an immediate response. For Notifications and Mailables, implement the `ShouldQueue` interface on the class itself — no separate Job needed.

### 2. Queueable Classes Still Running Synchronously

Scan listeners, notifications, and mailables for work that should be queued but still runs synchronously.

**Why it matters:** In Laravel, queues are not only about explicit Job classes. Event listeners, notifications, and mailables often contain the real I/O work. If these classes stay synchronous, the application still blocks requests even when the surrounding code looks "queue-aware".

**What to flag:**
- Event listeners doing slow work but not implementing `ShouldQueue`
- Notifications or mailables used in user-facing flows that do not implement `ShouldQueue`
- Listeners, notifications, or mailables making external API calls, heavy queries, or file operations synchronously
- Do NOT flag tiny listeners or mailables that do trivial in-memory work

**Suggestion:** Implement `ShouldQueue` on listeners, notifications, or mailables that perform slow or failure-prone work. Keep simple synchronous classes synchronous when the cost is truly negligible.

### 3. Non-Idempotent Jobs

Scan Job classes for operations that would produce incorrect results if the job runs more than once (retries after partial failure).

**Why it matters:** Queue jobs can fail partway through and be retried. If the job already performed a side effect (updated a record, sent a notification, charged a payment), retrying without checking causes duplicate actions, incorrect data, or double charges. This is especially dangerous because queue jobs run asynchronously — the data state may have changed between dispatch and execution.

**What to flag:**
- Model updates without checking current state first (e.g., `$model->update(['status' => 'processed'])` without verifying it is not already processed)
- Sending notifications or emails without checking if they were already sent
- Financial operations (charges, credits, transfers) without idempotency keys or pre-checks
- File operations (create, move, delete) without existence checks
- Multiple sequential operations where a failure after step 2 of 4 would leave step 1 and 2 already committed on retry

**Suggestion:** Make jobs idempotent — safe to run multiple times with the same result. Check current state before mutating: `if (is_null($message->read_at)) { ... }`. For critical operations, use database transactions or idempotency keys. Consider whether the model might be deleted before the job runs — use `#[DeleteWhenMissingModels]` when supported by the Laravel version, or pass the ID and look it up with a null check.

### 4. Missing Failure Handling for Critical Jobs

Scan Job classes for missing retry configuration, retry delay, timeout, or failure callbacks where the job's behavior makes those safeguards necessary.

**Why it matters:** Some jobs can safely fail and be retried with worker defaults. Others interact with flaky external systems, perform expensive processing, or need cleanup and alerting. Those jobs need explicit queue behavior to avoid silent failures or pointless immediate retries.

**What to flag:**
- Jobs calling external APIs or third-party services with no explicit retry/backoff strategy
- Jobs performing critical side effects with no `failed()` handling, alerting, or cleanup
- Jobs likely to hit timeouts with no `$timeout` and no clear timeout strategy
- Jobs relying entirely on global worker defaults when the job clearly has different reliability requirements from the rest of the queue
- Do NOT flag simple, non-critical jobs just because `$tries`, `$backoff`, or `failed()` are absent

**Suggestion:** Add `#[Tries(3)]` (or `$tries = 3`) when the job needs its own retry policy. Add `#[Backoff(60, 120, 300)]` or `backoff()` for external API calls where retrying immediately will likely fail again. Implement a `failed(Throwable $exception)` method when operations require alerting or cleanup. For jobs with timeouts, set `$timeout` and `$failOnTimeout = true` so failures are explicit rather than silent kills.

### 5. Fat Jobs (Jobs Doing Too Much)

Scan Job classes for jobs that handle too many responsibilities or process too much data in a single run.

**Why it matters:** A single job processing thousands of records risks timeout, memory exhaustion, and all-or-nothing failure. If it fails at record 500 of 1000, all work is lost. Breaking into smaller jobs enables parallel processing, better failure isolation, and smaller payloads.

**What to flag:**
- Job `handle()` methods longer than ~30 lines of logic
- Jobs that loop over large collections to perform individual operations (e.g., generating invoices for all users in one job)
- Jobs that perform 3+ distinct unrelated operations in sequence
- Jobs with `$timeout` set to very high values (300+ seconds) as a workaround for being too slow
- Jobs that query large datasets into memory without chunking

**Suggestion:** Break large jobs into smaller per-item jobs. Instead of one job generating 1000 invoices, dispatch 1000 individual `GenerateInvoice` jobs — each processes one record, runs in seconds, and can be retried independently. Use `Bus::batch()` to group them if you need to track overall progress. Use `Bus::chain()` if the operations must be sequential. For large datasets, query with `chunk()` or `cursor()` and dispatch a job per chunk.

### 6. Bloated Job Payloads and Stale Serialized Models

Scan job constructors and dispatch sites for Eloquent models being serialized with loaded relationships or unnecessary payload data.

**Why it matters:** When an Eloquent model is passed to a job constructor, Laravel serializes it. If the model has loaded relationships, those relationships are serialized too — bloating the job payload stored in the database. Worse, the serialized relationships become stale: the data was accurate at dispatch time but may have changed by the time the job runs.

**What to flag:**
- Jobs dispatched with models that were clearly eager-loaded just before dispatch
- Job constructors receiving full models when only an ID or a few scalars are actually needed
- Jobs carrying arrays or DTO-like payloads that include far more data than the job uses
- Jobs where models are passed with eagerly loaded relationships (e.g., dispatched after `$user = User::with('orders')->find($id)`)
- Do NOT flag every model parameter automatically — focus on actual serialization risk or stale-data risk

**Suggestion:** Add `#[WithoutRelations]` to model parameters when the job intentionally accepts a model instance and the Laravel version supports it. Otherwise, pass only the model's ID and look it up inside `handle()`. Prefer the smallest payload that still keeps the job readable and robust.

### 7. Duplicate Jobs Not Prevented

Scan for job dispatch points where the same job could be dispatched multiple times for the same entity, and for Job classes that should implement `ShouldBeUnique` but do not.

**Why it matters:** Users double-clicking buttons, network retransmissions, or multiple event triggers can queue duplicate jobs. Without uniqueness constraints, users receive duplicate emails, records are processed multiple times, or APIs are called repeatedly.

**What to flag:**
- Jobs dispatched from form submissions, webhook handlers, or event listeners without `ShouldBeUnique`
- Jobs implementing `ShouldBeUnique` but missing `uniqueId()` — the default lock key is the class name, meaning only ONE job of that class can be queued globally, blocking unrelated dispatches for different entities
- Jobs implementing `ShouldBeUnique` with no `#[UniqueFor()]` attribute or `$uniqueFor` property — lock never expires if the job fails silently
- Do NOT flag jobs where duplicates are harmless (e.g., cache warming, cleanup tasks)

**Suggestion:** Implement `ShouldBeUnique` on jobs where duplicate execution causes problems. Always override `uniqueId()` to return an entity-specific identifier (e.g., `return (string) $this->user->id`) so the lock is per-entity, not per-class. Set `#[UniqueFor(seconds)]` to ensure the lock expires. Use `ShouldBeUniqueUntilProcessing` instead if you want to allow a new dispatch once the current job starts processing.

### 8. Missing or Misused Job Middleware

Scan Job classes for concurrency or rate-limit problems that Laravel job middleware would solve more safely than ad hoc conditionals.

**Why it matters:** Some queue bugs are not about retries or uniqueness. They are about overlapping work, third-party rate limits, or repeated exceptions. Laravel job middleware exists for these cases and is often cleaner than writing custom locking or retry code inside `handle()`.

**What to flag:**
- Jobs that should not run concurrently for the same entity but use no `WithoutOverlapping` middleware and no equivalent locking
- Jobs hitting rate-limited APIs with no `RateLimited` middleware, throttling strategy, or explicit backoff logic
- Jobs that repeatedly fail due to transient third-party issues with no `ThrottlesExceptions` or equivalent protection
- Jobs with custom "skip if condition" logic that would be clearer as `Skip` middleware
- Do NOT insist on middleware when the job already has a clear, correct alternative

**Suggestion:** Use job middleware when it matches the problem well: `WithoutOverlapping` for per-entity mutual exclusion, `RateLimited` for third-party quotas, `ThrottlesExceptions` for noisy transient failures, and `Skip` for conditional no-op processing. Prefer the simplest clear solution, not middleware for its own sake.

### 9. No Queue Priority Separation

Scan job dispatch calls and queue configuration for all jobs being pushed to the same default queue.

**Why it matters:** When all jobs use the `default` queue, a burst of low-priority jobs (report generation, analytics) blocks time-sensitive jobs (password reset emails, payment confirmations). Priority queues ensure critical jobs are processed first.

**What to flag:**
- Projects with 5+ different Job classes all dispatching to the `default` queue (no `->onQueue()` calls, no `$queue` property on jobs)
- Mix of clearly time-sensitive jobs (auth emails, payment processing) and clearly deferrable jobs (reports, exports, cleanup) on the same queue
- Queue workers started without `--queue` flag specifying priority order
- Do NOT flag small projects with only 1-3 job types — queue separation adds complexity without benefit

**Suggestion:** Assign critical jobs to a `priority` queue using `->onQueue('priority')` at dispatch or by setting `public $queue = 'priority'` on the job class. Start workers with `--queue=priority,default` so priority jobs are processed first. For complex setups, use separate Supervisor/Horizon worker groups per queue with different `numprocs` allocations.

### 10. Missing Queue Tests

Scan the test directory for queue-related test coverage.

**Why it matters:** Queue jobs run asynchronously in production — they are invisible during manual testing. Without automated tests verifying that the right jobs are dispatched and that job logic works correctly, bugs surface only in production. Laravel provides dedicated testing tools (`Queue::fake()`, `Bus::fake()`, `withFakeQueueInteractions()`) that make queue testing straightforward.

**What to flag:**
- Projects with Job classes but no tests using `Queue::fake()`, `Bus::fake()`, or `withFakeQueueInteractions()`
- Job classes with complex `handle()` logic (conditionals, multiple operations) but no unit tests calling `handle()` directly
- Chain/batch dispatches (`Bus::chain()`, `Bus::batch()`) with no tests asserting correct structure (`Bus::assertChained()`, `Bus::assertBatched()`)
- Jobs using `release()` or `Skip` middleware with no tests verifying correct skip/release behavior
- Do NOT flag projects with no tests at all — that is a broader issue, not queue-specific

**Suggestion:** Use `Queue::fake()` to verify correct jobs are dispatched from controllers and event listeners. Use `Bus::fake()` with `Bus::assertChained()` or `Bus::assertBatched()` for chain/batch verification. Use `withFakeQueueInteractions()` on individual job instances to test internal logic — assert `$job->assertReleased()`, `$job->assertDeleted()`, etc. Note: `Queue::fake()` does NOT intercept `Bus::chain()` or `Bus::batch()` — use `Bus::fake()` for those.

### 11. Incorrect `afterCommit` Usage

Scan for jobs dispatched inside database transactions without `afterCommit()`.

**Why it matters:** When a job is dispatched inside a database transaction, the job may be picked up by a worker before the transaction commits. The worker then queries for data that does not exist yet, causing failures. If the transaction rolls back, the job was already queued for data that was never persisted.

**What to flag:**
- `DB::transaction(function () { ... Job::dispatch() ... })` patterns without `->afterCommit()`
- Jobs dispatched in model observers (`creating`, `created`, `updating`, etc.) without `$afterCommit = true` on the observer or `->afterCommit()` on the dispatch — model events fire inside the transaction
- `config/queue.php` with `'after_commit' => false` (the default) in projects that frequently dispatch jobs inside transactions
- Do NOT flag dispatches clearly outside any transaction context

**Suggestion:** Add `->afterCommit()` to job dispatches inside transactions: `Job::dispatch($model)->afterCommit()`. This ensures the job is only queued after the transaction successfully commits. Alternatively, set `'after_commit' => true` in `config/queue.php` to make this the default for all dispatches. For dispatches in model observers, ensure the observer or job is configured for after-commit behavior.

### 12. Jobs Not Using `Batchable` Trait When Batched

Scan for jobs dispatched via `Bus::batch()` that do not use the `Batchable` trait.

**Why it matters:** Without the `Batchable` trait, a job inside a batch cannot check if the batch was cancelled (`$this->batch()->cancelled()`), cannot access batch metadata, and cannot properly report its status back to the batch. The batch loses visibility into individual job status.

**What to flag:**
- Job classes dispatched via `Bus::batch([...])` that do not `use Batchable`
- Jobs using `Batchable` but never checking `$this->batch()->cancelled()` inside `handle()` — if the batch is cancelled, the job continues processing unnecessarily

**Suggestion:** Add `use Batchable` to any job that will be dispatched as part of a batch. Check `if ($this->batch()->cancelled()) { return; }` at the start of the `handle()` method to respect batch cancellation.

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

- **High** — Clear production bugs or reliability hazards: non-idempotent critical jobs, jobs dispatched inside transactions without `afterCommit`, long-running synchronous operations blocking HTTP responses, queueable classes doing slow work synchronously
- **Medium** — Meaningful reliability and maintainability issues: fat jobs risking timeouts, missing retry/backoff on fragile integrations, no uniqueness on duplicate-prone jobs, overlapping/rate-limited jobs with no proper coordination, all jobs on the same queue despite obvious priority differences
- **Low** — Context-dependent improvements: missing queue tests, batch ergonomics, payload-size optimizations

### What's Done Well

End with a short section acknowledging patterns the project already follows correctly. This prevents the audit from feeling like a list of complaints and validates good decisions.

## Important Guidelines

- Do NOT flag projects that do not use queues at all — this audit is for projects that already have queue/job infrastructure.
- Do NOT nitpick simple jobs. A 5-line job that sends a single email is fine without `failed()`, `$backoff`, or `ShouldBeUnique`.
- Do NOT flag every job without `ShouldBeUnique` — only flag where duplicates would actually cause problems.
- Do NOT treat the absence of `$tries`, `$backoff`, `failed()`, middleware, or `#[WithoutRelations]` as a bug by itself. Flag them only when the job's behavior makes the missing feature matter.
- DO read enough of each file to understand context before flagging. A job with a long `handle()` might contain comments, not logic.
- DO check which Laravel version the project uses (`composer.json`) — some features like `#[Tries()]`, `#[Backoff()]`, `#[DeleteWhenMissingModels]`, and `Skip` middleware are only available in recent versions.
- DO prefer repository-verifiable findings. If production infrastructure is not represented in the codebase, say that it cannot be verified from the repository rather than guessing.
- DO consider the project size. A project with 2 simple jobs does not need priority queues, Horizon, or batch processing.
- Present findings as suggestions, not mandates. Queue architecture depends on scale, infrastructure, and team preferences.
