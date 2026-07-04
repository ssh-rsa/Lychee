# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Lychee is a self-hosted photo-management system: Laravel 12 (PHP 8.4) API backend + Vue3/TypeScript SPA frontend. A separate Python sidecar (`ai-vision-service/`, FastAPI) provides facial recognition over a shared Docker volume.

## Full workflow rules

[AGENTS.md](AGENTS.md) defines the mandatory Specification-Driven Development workflow (spec → plan → tasks under `docs/specs/4-architecture/features/`, open-questions log, ADRs, commit protocol). Read it before planning or implementing non-trivial work — it is not optional and is not repeated here.

[docs/specs/4-architecture/knowledge-map.md](docs/specs/4-architecture/knowledge-map.md) is the living architecture reference (modules, request flows, recent feature designs). Skim it before touching unfamiliar areas, and update it when you add modules/contracts.

[docs/specs/3-reference/coding-conventions.md](docs/specs/3-reference/coding-conventions.md) has the full PHP/Vue3/testing conventions; key rules are summarized below.

## Commands

### Backend (PHP)
- Run all tests: `php artisan test` (migrations auto-apply to the SQLite test DB)
- Run a single test: `php artisan test --filter=TestName`
- Run a specific suite: `vendor/bin/phpunit --testsuite Unit` (suites: `Unit`, `Feature_v2`, `Install`, `ImageProcessing`, `Precomputing`, `Webshop`, `AssistedVision`; see `Makefile` `test_*` targets)
- Static analysis: `make phpstan` (level 6 minimum; must be clean before committing)
- Code style: `vendor/bin/php-cs-fixer fix -v --config=.php-cs-fixer.php` (or `make formatting`, which also runs Rector)
- Generate TS types from PHP DTOs: `make gen_typescript_types`
- Unused-class check: `make class-leak`

### Frontend (Vue3/TypeScript)
- Dev server: `npm run dev`
- Build: `npm run build`
- Type check: `npm run check`
- Lint: `npm run lint`
- Format: `npm run format` (check-only: `npm run check-formatting`)

### Full quality gate before considering work done
PHP changes: `vendor/bin/php-cs-fixer fix` → `php artisan test` → `make phpstan`.
Frontend changes: `npm run format` → `npm run check`.
Run both sequences if both stacks changed.

## CRITICAL: Database safety

Never run `php artisan migrate:fresh`, `migrate:reset`, `db:wipe`, or any other reset/wipe command. The test suite uses a separate SQLite DB (`database/database.sqlite`, configured in `phpunit.xml`), but the main app typically points at MySQL/PostgreSQL — migration-reset commands run without care will hit the real dev/production database. Tests use the `DatabaseTransactions` trait, so no manual cleanup is needed between runs. To reset only the test DB: delete and recreate `database/database.sqlite`, then run tests (migrations reapply automatically).

## Architecture

### Backend layout (`app/`)
Standard layered Laravel structure: `Http/Controllers` (routing) → `Http/Requests` (validation) → `Actions`/`Services` (business logic) → `Models` (Eloquent) → `Http/Resources` (response transform, extend **Spatie Data**, not `JsonResource`). Routes live in `routes/api_v2.php`, `routes/api_v2_shop.php`, `routes/web_v2.php`, `routes/web-admin-v2.php` — there are no Blade views, the frontend is entirely Vue3.

Other notable directories: `Actions/` (single-responsibility command objects, including the photo-upload pipe pipeline under `Actions/Photo/Pipes/`), `DTO/` (Spatie Data objects), `SmartAlbums/` (virtual albums like Recent/Highlighted, distinct from regular nested-set `Album` rows), `Jobs/`/`Events/`/`Listeners/` (async work, e.g. album stats recomputation cascades from a changed photo/album up to the root), `Enum/` (Spatie enums).

Request convention: in `Http/Requests`, `$this->user` is the authenticated requester; `$this->user2` is a user object supplied via the query (a different user being acted upon).

### Frontend layout (`resources/js/`)
Vue3 Composition API + TypeScript, PrimeVue components, Pinia stores, Vue Router. `services/` holds axios wrappers (base URL via `${Constants.getApiUrl()}`). `layouts/` implements the photo grid algorithms (square/justified/masonry/grid). `composables/` holds reusable composition functions. Two Vite builds exist: the main app (`vite.config.ts`) and an embeddable widget bundle (`vite.embed.config.ts`, `npm run build:embed`).

### Money
Monetary values (webshop/e-commerce) use `moneyphp/money` and are always stored as integers in the smallest currency unit (e.g. $10.99 → `1099`) — never floats.

### Translations
Source of truth is `lang/<locale>/*.php` (snake_case keys, nested arrays for grouping). Never touch `lang/php_*.json` — those are auto-generated and untracked.

### AI Vision sidecar
Lychee and the Python face-detection service communicate over REST + webhook, sharing a Docker volume for file access (no HTTP file transfer) and a symmetric API key (`AI_VISION_FACE_API_KEY` / `VISION_FACE_API_KEY`) via `X-API-Key`. Lychee dispatches scan jobs, the Python service posts results back to a Lychee callback endpoint.

## Coding conventions (PHP)
- snake_case variables, PSR-4 classes, strict comparison (`===`), no `empty()`, `in_array()` always with the third (`true`) argument, no code duplication across if/else branches, only booleans in `if` conditions.
- New files: license header + one blank line after `<?php`.

## Coding conventions (Vue3)
- Composition API + TypeScript only.
- No `async`/`await`; use `.then()`.
- Named function declarations (`function foo() {}`), not `const foo = () => {}`.
- Single-file component block order: `<template>`, then `<script lang="ts">`, then `<style>`.

## Testing conventions
- `tests/Unit/*` extends `AbstractTestCase`; `tests/Feature_v2/*` extends `BaseApiWithDataTest`.
- Never mock the database — the in-memory/file SQLite test DB is used directly.
