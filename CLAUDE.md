# laravel-jetstream-app

Laravel 11 + Jetstream 5 (Inertia/Vue 3) recipe on Zerops with PostgreSQL, KeyDB/Redis, S3-compatible object storage, and Mailpit — shared `base` setup extended by `prod` and `dev`.

## Zerops service facts

- HTTP port: `80` (Nginx + PHP-FPM; custom `site.conf.tmpl`)
- Siblings: `db` (PostgreSQL), `redis` (KeyDB), `storage` (Object Storage), `mailpit` (SMTP) — env vars follow the `${hostname_*}` pattern (`DB_*`, `REDIS_HOST/PORT`, `AWS_*` for storage, `MAIL_*` for mailpit)
- Runtime base: `php-nginx@8.4` (build on `php@8.4` + `nodejs@22`; dev runtime installs Node 22 via `prepareCommands`)

## Zerops dev (hybrid)

Runtime (`php-nginx`) auto-serves PHP changes immediately — edit `.blade.php` / `.php` and they take effect on the next request.

**The `dev` setup intentionally skips `npm run build` in `buildCommands`** to keep the HMR workflow (`npm run dev`) fast. Consequence: `public/build/manifest.json` does NOT exist after a fresh deploy, so any view using `@vite(...)` (or the Inertia/Vue page entrypoints) throws HTTP 500 `Vite manifest not found` on the first request.

**After every `zerops_deploy` on this service and BEFORE running `zerops_verify`**, do ONE of:

- **One-shot build** — `ssh appdev 'cd /var/www && npm run build'`. Creates the manifest; survives until the next deploy rsyncs the build tree back in.
- **Long-running dev server (HMR)** — `ssh appdev 'cd /var/www && nohup npm run dev > /tmp/vite.log 2>&1 &'`. Drops `public/build/hot`; Laravel's `@vite` helper routes asset URLs to the dev server. Containers restart on every `zerops_deploy`, so rerun after each redeploy.

**Do NOT add `npm run build` to dev `buildCommands`** — it adds ~20–30 s to every push and defeats the HMR-first design.

**All platform operations (deploy, env / scaling / storage / domains) go through the Zerops development workflow via `zcp` MCP tools. Don't shell out to `zcli`.**

## Notes

- No dedicated `worker` setup in `zerops.yaml` — queue jobs process inline via `QUEUE_CONNECTION: redis` + Jetstream defaults. If jobs need a dedicated consumer, add a separate service running `php artisan queue:work`.
- Dev installs Node 22 on the runtime container via `prepareCommands` (NodeSource apt setup), then `initCommands` runs `zsc scale ram +0.5GB 10m`, `composer install`, and `npm install` — dev container is ready to `npm run dev` immediately over SSH.
- Artisan caches (`view:cache`, `config:cache`, `route:cache`, `optimize`) and `migrate --isolated --force` run in prod `initCommands`, not `buildCommands` — the build path differs from the runtime path.
- Before `php artisan down`, run `zsc health-check disable` and re-enable after `php artisan up`, otherwise Zerops will kill the container during the 5-minute retry window.
