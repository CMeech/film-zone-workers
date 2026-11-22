# Game creation CSP fix

Replaced nested `newGame.*` x-model bindings with top-level component fields.
The request retains the existing game field names, CSRF header, and same-origin
credentials. All four fields reset after successful creation.

## Verification

Executed on 2026-09-13 through Docker Compose:

- `PLAYWRIGHT_SCRIPT='test:browser -- create-game.spec.js' docker compose -f docker-compose.browser.yml up --build --abort-on-container-exit --exit-code-from browser-tests`: reproduced Alpine's `Property assignments are prohibited in the CSP build` on both viewports before the fix (2 failing tests).
- `docker compose run --rm --no-deps --build -e NODE_ENV=production bundler sh -c 'npm run build-esbuild && npm run build-tailwind'`: rebuilt generated assets successfully.
- `docker compose -f docker-compose.browser.yml up --abort-on-container-exit --exit-code-from browser-tests`: 28 passed, 4 expected viewport-specific skips. New regression exercises login, explicit Varsity team selection, `/games/list`, all four inputs, `/games/create` request payload, reset, disabled state, and visibility after reload at 390x844 and 1440x900. No console errors or uncaught page errors in the new regression.
- `docker compose -f docker-compose.test.yml up --build --abort-on-container-exit --exit-code-from app-tests`: 51 passed, 1 unrelated failure, 17 limiter warnings. `test_tests_use_explicit_test_database_even_when_run_tests_is_false_string` expects `stats-data/stats-test.db`, but configuration returns the supplied `custom.db`. No backend/configuration code changed.
- `git diff --check`: passed.

Initial setup runs lacked event options because the default active team had no
seeded events. The regression now selects Varsity explicitly and fixes the
browser date to the seeded calendar period. A subsequent run exposed stale
local generated bundles overwriting Docker's build-stage output during `COPY . .`;
rebuilding local generated assets resolved verification. That Docker packaging
issue is separate and was not changed here.

Reviewed application logs for the successful browser run: no application errors;
the existing in-memory Flask-Limiter warning remains. Build output also reports
the existing CSS `@property` warning, GSAP package-condition diagnostics, npm
install-script warnings, and 6 dependency audit findings (1 low, 3 high, 2 critical).

`docker compose -f docker-compose.browser.yml down --volumes` removed disposable
browser containers, database, resources, dependency volume, and network. Existing
development containers and pytest volume were preserved. Runtime database and
uploads are unchanged. No production deployment performed.
