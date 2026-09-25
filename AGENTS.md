# AGENTS.md

## What this is

Docker-based dev environment for a **Laravel 13 fullstack** app (Livewire 4 + Flux UI 2 + Tailwind 4 + Vite 8) on **PostgreSQL 17**.

- Repo root is the orchestration layer (`docker-compose.yml`, root `.env`). The Laravel app lives entirely in `backend/`.
- The root `README.md` is a stub (`# APM-ENG-TEMPLATE`). The real spec is **`backend/README.md`** — architecture, Clean Code rules and the pre-commit checklist. Read it before touching app code.

## Layout and boundaries

Target Clean Architecture (`backend/README.md`):

- `backend/app/Core/<Context>/` — business rules: `Entities`, `ValueObjects`, `DTOs`, `Repositories` (interfaces), `Services`, `Exceptions`. **No Laravel, Eloquent, HTTP, facades or `Request` here.**
- `backend/app/Infrastructure/` — Eloquent models / repository implementations, external gateways.
- `backend/app/Http/` (Controllers, Requests, Resources) and `backend/app/Livewire/` — interface layer. `backend/app/Providers/` wires interface → implementation.
- Dependency rule: `Http`/`Livewire` → `Core`; `Infrastructure` → `Core`; `Core` imports nothing from the other layers.
- First context implemented: `Core/Catalog` (`Entities`, `ValueObjects`, `DTOs`, `Exceptions`, `Repositories` interfaces, `CategoryService`/`ProductService`), `Infrastructure/Persistence/Eloquent` (models + repository impls), and `App\Livewire\{ProductManager,CategoryManager}`. Repository interface → implementation bindings live in `AppServiceProvider::register()`. Create new contexts the same way only when needed — do not scaffold empty layers.
- The store front page is `/` (`resources/views/store.blade.php`) with a Produtos/Categorias tab switch. Free Flux has **no `flux:tabs`**, so tabs are plain Alpine (`x-data="{ tab: ... }"`) wrapping the two Livewire components.

## Commands

Run from the repo root unless noted.

- First-time setup: `cp .env.example .env && cp backend/.env.example backend/.env`. Set `UID`/`GID` in the root `.env` to `id -u` / `id -g` if not 1000.
- Start: `docker compose up -d --build` (first run does `composer install` in backend and `npm install` in vite; watch with `docker compose logs -f`).
- Init app: `docker compose exec backend php artisan key:generate` then `docker compose exec backend php artisan migrate`.
- Artisan: `docker compose exec backend php artisan <cmd>`. Composer: `docker compose exec backend composer <cmd>`.
- Logs: `docker compose logs -f <backend|vite|db>`.
- Formatting (required before commit): `docker compose exec backend ./vendor/bin/pint` (or `backend/vendor/bin/pint`).
- Host ports: app `8000`, Vite `5173`, Postgres `5433`. Inside Docker the DB is `db:5432`.
- `docker compose down` keeps the DB volume; `docker compose down -v` wipes it.

## Gotchas

- **Tests are intentionally absent.** There is no `backend/tests/` and no `phpunit.xml` (removed in commit `Use Core layer with Services, remove tests`). `composer test` / `php artisan test` will not work — do not re-add a test harness unprompted; verify with `pint` and manual checks instead.
- `backend/.env` uses `DB_HOST=db` (Docker network), so running `artisan`/`phpunit` directly on the host will not reach the DB.
- Use `declare(strict_types=1);` with typed parameters and returns (project convention).
- Never edit an applied migration — add a new one. Keep `$fillable` explicit. Secrets only in `.env` (gitignored).
- Commits: small, imperative message (e.g. `Add user creation to UserService`).
- Compose service names are `backend`, `vite`, `db`; container names are `dev-backend`, `dev-front`, `dev-db`.
