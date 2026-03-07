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

## 2. Choose Adminpanel and Other Packages

Quite often, Laravel projects need admin panel. My personal favorite is [Filament](https://filamentphp.com/), so I would install it right away, so that Laravel Boost would get its guidelines. After installing Filament, re-run `php artisan boost:install`.

Next, think about other fundamental Laravel packages that you may want to install.

You may consult AI agent with [this prompt](./prompts/choose-laravel-packages.md).

Install the packages you have chosen.

---