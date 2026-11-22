# Game video iframe CSP fix

Alpine CSP explicitly prohibits expression evaluation on iframe elements. Move
the reactive effect to the video wrapper and set the frame source in the
component method. Only HTTP(S) URLs are assigned; empty or incomplete input
removes the source. Existing server CSP restrictions remain in force. Game
script URLs retain their original form without cache-busting query parameters.

## Verification

Executed 2026-09-13 through Docker Compose:

- `PLAYWRIGHT_SCRIPT='test:browser -- game-video.spec.js' docker compose -f docker-compose.browser.yml up --build --abort-on-container-exit --exit-code-from browser-tests`: before the fix, 2 failed; the iframe never received its source. Installed Alpine source confirms the iframe evaluation prohibition.
- `docker compose -f docker-compose.browser.yml up --build --abort-on-container-exit --exit-code-from browser-tests`: after the fix, 30 passed, 4 expected viewport-specific skips. Assets rebuilt in the production image. Includes game creation and video save/reload regression at 390x844 and 1440x900. Initial video test used a stubbed YouTube response and a short-link URL.
- `docker compose run --rm --no-deps --build -e NODE_ENV=production bundler sh -c 'npm run build-esbuild && npm run build-tailwind'`: refreshed local generated assets successfully.
- `PLAYWRIGHT_SCRIPT='test:browser -- game-video.spec.js' docker compose -f docker-compose.browser.yml up --abort-on-container-exit --exit-code-from browser-tests`: 2 passed with the user's exact URL, `https://www.youtube.com/embed/EWdlDxxnZoY?si=EZXv1-RQhpJ6Bcqy`, without a YouTube response stub. Verified Save Changes POST succeeds, URL persists after reload, iframe source retains the query string, and clearing the field removes the source. No uncaught page errors or Alpine evaluation errors. Playback inside YouTube was not asserted.
- `git diff --check`: passed.

Successful application logs contain no application errors; existing Flask-Limiter
in-memory warning remains. Console checks target Alpine errors because seeded
game 501 initially contains an example.test URL blocked by the existing frame
CSP, and third-party video/network messages are outside this regression. Build
warnings remain for CSS @property, GSAP package conditions, npm install scripts,
and 6 npm audit findings. Backend code was unchanged; the prior backend run's
unrelated configuration-test failure remains recorded in game-create-csp.md.

Removed disposable browser containers and volumes using
`docker compose -f docker-compose.browser.yml down --volumes`. Existing development
containers, pytest volume, runtime database, and uploads were preserved. Changes
are local and require deployment of both templates and rebuilt JavaScript.
