## New Laravel Project with AI

Step-by-Step process of creating a new Laravel project with AI assistance.

---

## 1. Tech-Stack Choice BEFORE `laravel new`

Read the initial project description (*from client or yourself*) and decide the tech-stack:

Laravel with React.js/Vue.js/Livewire or Blade or API-only?

Based on that choice, you need to run `laravel new` and choose the **starter kit** (or `None`) accordingly.

In the installation wizard, choose to install [Laravel Boost](https://github.com/laravel/boost) and enable skills/guidelines.

After we have this new project folder, put the initial job description as Markdown file in `docs/project-description.md`. The rest of the workflow is based on that filename.

Initialize Git repository and commit this initial skeleton.

---

## 2. Filament Adminpanel

If your project needs admin panel, install it right away, as it's a fundamental architectural package. My personal favorite is [Filament](https://filamentphp.com/), so I would install it now, so that Laravel Boost would get its guidelines. After installing Filament, re-run `php artisan boost:install`.

Add Filament as a choice into `docs/project-description.md`, in the tech-stack choice section, if it's not mentioned there.

---

## 3. User Stories

Run [this user stories prompt](./prompts/prompt-user-stories.md) in Claude Code or other AI agent of choice. It should ask you questions to clarify the project details.

As a result, you should get a new file `docs/user-stories.md`. Review it with AI and/or manually before proceeding.

This step is **very important** for the project outcome, as the question asked by AI agent would help you clarify the choices for tech-stack and code structure, or discover something important before any code is written or packages chosen.

---

## 4. Third-Party Packages

Next, think about other Laravel packages that you may want to install, like `spatie/laravel-medialibrary` or similar.

You may consult AI agent with [this prompt](./prompts/prompt-choose-laravel-packages.md).

Install the packages you have chosen.

Run `php artisan boost:install` again to see if any packages have published their AI guidelines/skills for Laravel Boost.

Add those packages to the list of tech-stack in `docs/project-description.md`.

For Filament projects, I use and recommend my own package [FilaCheck](https://filamentexamples.com/filacheck) to auto-fix deprecated Filament code from older versions, after AI agent executed the prompt.

---

## 5. Custom AI Guidelines and MCPs

If you want to add custom guidelines for Laravel/Filament or other packages, put them now into `.ai/guidelines` folder as Markdown files and re-run `php artisan boost:install`. Read more in the [docs of Laravel Boost](https://laravel.com/docs/12.x/boost#adding-custom-ai-guidelines).

You can use my own guidelines for [Laravel](./guidelines/laraveldaily-laravel.md) and [Filament](./guidelines/laraveldaily-filament.md). The most important parts are to force AI agent to generate/run Pest tests, and to use [Context7](https://github.com/upstash/context7) as a fallback for docs in case of Laravel Boost failure (*that happens rarely*).

Also here, add any MCP servers you want to use. I personally don't use anything except for Context7, mentioned above.

---

## 6. User Stories to Project Phases

Run [this project phases prompt](./prompts/prompt-project-phases.md) in AI agent. As a result, you should get a new file `docs/project-phases.md`. Review it with AI and/or manually.

---


## 7. Start Prompting to Write Code

From here, you should be able to prompt something like `Work on Phase 1 of @docs/project-phases.md`, same for Phase 2, 3, and so on, as the descriptions of the phases are detailed enough already.

But, of course, if you see some details missing, you can course correct along the way.