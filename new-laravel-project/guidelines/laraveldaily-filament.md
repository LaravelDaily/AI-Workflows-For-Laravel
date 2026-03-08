## Filament Rules

- When generating Filament resource, you MUST generate Filament smoke tests to check if the Resource works. When making changes to Filament resource, you MUST run the tests (generate them if they don't exist) and make changes to resource/tests to make the tests pass.
- When writing tests with Pest, use syntax `Livewire::test(class)` and not `livewire(class)`, to avoid extra dependency on `pestphp/pest-plugin-livewire`.
- When generating Filament resource, don't generate View page or Infolist, unless specifically instructed.

---

## Auto-Run FilaCheck

- If package `laraveldaily/filacheck` or `laraveldaily/filacheck-pro` is installed, and if you have created/modified any files in `app/Filament` folder, you must run `vendor/bin/filacheck --fix` before running Pest tests, to ensure there is no deprecated Filament code.