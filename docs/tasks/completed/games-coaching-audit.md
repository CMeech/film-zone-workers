# Games coaching audit

Audited on 2026-09-13 after the game-creation and video iframe CSP repairs.
Scope: games templates, Alpine component state and event handlers, metadata,
stat persistence, and access checks for game mutations.

## Repairs

- Rotation fields no longer use nested/indexed x-model assignments. Explicit
  input handlers update numeric entries in the component; blank/invalid numbers
  become null, preserving the game JSON shape.
- Team tabs use a component method rather than an inline assignment.
- Total is a read-only aggregate: selecting it resets editing and hides match
  controls instead of opening a sixth-set rotation editor.
- Only coaches/admins receive match-control markup. Players retain the stats
  viewer. Existing desktop-only match controls are preserved.
- Opponent numbers outside the integer range 1–99 cannot be added.
- Wrong-team metadata update/delete responses serialize the error string,
  preserving their existing 401 rejection instead of throwing a JSON error.
- Preserved the prior Dockerfile correction: source is copied before generated
  assets, preventing stale local bundles from overwriting the image build.
  Script URLs remain unversioned as requested.

## Workflow coverage

| Workflow | Evidence |
| --- | --- |
| Create game | Existing mobile/desktop form test; new coach creates game with only required opponent field |
| Rotations | All 12 positions edited, autosaved, reloaded, and checked against server JSON |
| Opponent roster | Invalid zero disabled; player 20 added and used for stats |
| Numeric stats | Blocks, digs, aces, missed serves, errors recorded and persisted |
| Positional stats | Kill target and set from/to recorded and persisted |
| Scores | Team increment, opponent increment/decrements, zero floor |
| Sets and totals | Separate first/second set stats, aggregate aces, opponent/team tabs, Total controls hidden |
| Metadata/video | Final score and exact supplied YouTube embed URL survive reload without losing event 402 |
| Delete | Created game deleted through confirmation and removed from list |
| Player viewing | All five set tabs, Total, opponent/team tabs at mobile and desktop; no coaching controls |
| Access/schema | Anonymous, player, wrong-team, valid coach across all three existing-game mutation routes; invalid stat document rejected |

## Verification

All project tools ran through Docker Compose.

- `PLAYWRIGHT_SCRIPT='test:browser -- game-coaching.spec.js' docker compose -f docker-compose.browser.yml up --build --abort-on-container-exit --exit-code-from browser-tests`: baseline 1 failed, 1 skipped. All twelve rotation input events produced `Property assignments are prohibited in the CSP build`. After the first fix, 1 passed, 1 skipped, including all stat types and actual persisted JSON.
- `docker compose -f docker-compose.browser.yml up --build --abort-on-container-exit --exit-code-from browser-tests`: final expanded suite **33 passed, 5 skipped** (38 cases). Four pre-existing viewport skips plus desktop-only match editing skipped at mobile. Viewports 390x844 and 1440x900; routes include login, dashboard, team selection, `/games/list`, and `/games/view/<id>`. No page/console errors in the full coach workflow. Player test checks uncaught page errors. Existing YouTube test filters Alpine console errors, since external-frame messages are outside the application regression.
- `PLAYWRIGHT_SCRIPT='test:browser -- game-video.spec.js' docker compose -f docker-compose.browser.yml up --abort-on-container-exit --exit-code-from browser-tests`: **2 passed** after expanding final-score and linked-event assertions. Playback inside YouTube is not asserted.
- `docker compose -f docker-compose.test.yml up --build --abort-on-container-exit --exit-code-from app-tests`: first run 61 passed, 4 failed. Three were new test setup failures (the shared test app resolves templates under tests/); setting the real template directory fixed those. One was the previously reported configuration-test failure.
- `docker compose -f docker-compose.test.yml run --rm --no-deps app-tests tests/games -q`: **13 passed**, 13 limiter warnings. One earlier invocation accidentally supplied `pytest` twice, ran zero tests, and was corrected.
- `docker compose -f docker-compose.test.yml run --rm --no-deps app-tests`: **64 passed, 1 failed**, 30 limiter warnings. Remaining failure: `test_tests_use_explicit_test_database_even_when_run_tests_is_false_string` expects stats-test.db despite explicit custom.db. Configuration is unchanged.
- `docker compose run --rm --no-deps --build -e NODE_ENV=production bundler sh -c 'npm run build-esbuild && npm run build-tailwind'`: rebuilt local generated JS/CSS successfully. Browser image also rebuilt assets.
- `git diff --check`: passed.

Reviewed successful browser application logs: no application errors. Existing
warnings: in-memory rate limiter, CSS @property, GSAP package conditions, npm
install scripts, six npm audit findings (one low, three high, two critical), and
Compose notices about preserved unrelated containers.

## Separate follow-ups and limits

These source findings are outside the CSP interaction repairs and remain unchanged:

- Event IDs supplied directly to game create/update are not checked against the
  active team before linking. The normal UI lists active-team events, but the
  API should validate ownership independently.
- Creation retrieves MAX(id) in a separate query instead of the inserted row's
  ID on one connection; concurrent creation can return the wrong game.
- Offline cache/concurrent-editor behavior remains last-write-wins. This audit
  verifies successful online saves, not recovery from connectivity loss.
- Shared wrong-role authorization currently renders an HTML error with status
  200; tests verify writes are denied without changing that global contract.
- Mobile match controls remain hidden by the existing responsive design. Mobile
  game creation, metadata, and viewing are covered; mobile stat entry is not enabled.

## Cleanup and deployment

Disposable browser containers and volumes are removed with
`docker compose -f docker-compose.browser.yml down --volumes`. Existing development
containers and pytest volume are preserved. No runtime database/upload changes.
No production deployment: ship updated templates, rebuilt JavaScript, and the
backend error-handler fixes together.
