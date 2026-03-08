Plan this project into phases and what needs to be built. I want to have a complete task list that needs to happen, based on `@docs/user-stories.md` and `@docs/project-description.md`.

For each task, specify the automated feature tests to be generated, as acceptance criteria.

For phases and sub-phases, use numeration like "Phase 1" or "Phase 5.3" so they could be referenced by numbers, when later given to AI agent to implement.

Check in the code, which tasks are already completed, and mark them accordingly.
Put the result into `@docs/project-phases.md`.

---

## Phase 1: Database Structure

The first phase of that file should be about generating DB structure in the form of models/migrations/factories/seeders. Generate the structure instructions, according to the requirements for the `@docs/user-stories.md` and `@docs/project-description.md`.

Follow best practices and typical convensions for Laravel 12 databases.

### Guidelines for DB structure: 

- Don't use Enum DB fields, unless absolutely sure the values won't change. For values that would rarely change, create string DB value with a comment of first values, so that PHP Enum is expected in the future. For values that would change more often, create lookup tables with foreign keys.
- Don't create new DB tables that may conflict with same name as existing default Laravel DB tables, like "users" or "jobs".
- For file/image uploads, use the package `spatie/laravel-medialibrary` with DB table `media` coming from that package, unless instructed otherwise.

---

## Technical Detail

If project uses Livewire starter kit, then create new pages as Laravel Controllers and not as Livewire components, unless specifically stated otherwise or unless dynamic Livewire behavior is required on that page.