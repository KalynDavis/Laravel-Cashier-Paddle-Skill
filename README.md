# Laravel Cashier Paddle Boost Skill

Generated from the Laravel 13.x Cashier Paddle documentation at:

https://laravel.com/docs/13.x/cashier-paddle

This repository converts the original agent markdown into a Laravel Boost-compatible agent skill for Cashier Paddle 2.x and Paddle Billing.

## Files included

- `.ai/skills/laravel-cashier-paddle-development/SKILL.md` — local Boost custom skill path for immediate use in an application.
- `resources/boost/skills/laravel-cashier-paddle-development/SKILL.md` — package-distributed Boost skill path. If this repository is turned into a Composer package, Boost can discover and install this skill from here.
- `AGENTS.md` — legacy root-level coding-agent instructions for Laravel Cashier Paddle billing work.
- `docs/agents/laravel-cashier-paddle-13.md` — detailed implementation reference.
- `docs/agents/laravel-cashier-paddle-checklist.md` — implementation/review checklist.

## Local app installation

Copy the `.ai/skills/laravel-cashier-paddle-development` directory into the root of a Laravel application, then run:

```bash
php artisan boost:update
```

Laravel Boost installs custom skills from `.ai/skills/{skill-name}/SKILL.md`.

## Package installation

To distribute this skill from a Composer package, keep the skill at:

```text
resources/boost/skills/laravel-cashier-paddle-development/SKILL.md
```

After the package is installed in a Laravel app, run:

```bash
php artisan boost:install
```

or:

```bash
php artisan boost:update --discover
```

Boost can then install the package skill based on the user's selected Boost features.

## Legacy markdown usage

1. Copy `AGENTS.md` to the root of a Laravel repository.
2. Copy `docs/agents/*` into the repository as deeper billing context.
3. Customize price catalog names, routes, model names, and domain-specific concepts such as `Team`, `Order`, `License`, or `Entitlement`.
4. Keep the docs source URL in the files so future agents can refresh against current docs.
