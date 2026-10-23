# Working on this repo in the Base44 sandbox

## What it is

Spires of Azeroth is a **static, build-free** game: vanilla ES modules + JSON data under
`spiresofazeroth/`, with dev tools in `tools/spires/`. There is no bundler, no package
manager, no database and **no environment variables or credentials**.

In the sandbox it runs as a single nginx service (`docker-compose.base44.yml`) serving this
checkout from `/srv/repo` on host port 3000:

- `http://localhost:3000/spiresofazeroth/` — the game (`/` redirects there, which is what the preview loads)
- config: `.base44/nginx.conf` (sandbox-only; the repo has no other compose file)

Edits to the checkout are served as-is — nginx reads the bind-mounted files, so there is no
watcher and no rebuild. After changing front-end source or data, reload the preview. Bump
`spiresofazeroth/js/version.js` `BUILD` when you change `data/` or module URLs, as the game
cache-busts every data fetch with it.

## Art, audio and vendor files come from the live site

Blizzard icons/models/sounds and the shared TheWoWDB chrome + Three.js vendor copy are **not
in this repository** (see `.gitignore`). `.base44/nginx.conf` mirrors
`tools/spires/dev-server.mjs`: `/assets/**`, `/media/sounds/**`, `/spiresofazeroth/assets/**`
and `/static/**` are tried in the checkout first and fetched from `thewowdb.com` otherwise.
That needs outbound network; without it the game still plays, just without art/audio.

`/api/**` is deliberately **not** proxied: the in-game bug-report form then falls back to its
GitHub issues link instead of sending mail from the sandbox.

One exception: the shared chrome scripts the page loads from the live site probe `/api/auth/me`
to paint their account label. This checkout has no auth backend, so `.base44/nginx.conf` answers
that single path with the chrome's own anonymous state (`{"user":null}`) — a 404 there only spams
the browser console on every load. Nothing else under `/api/` is answered.

The repo's own `tools/spires/dev-server.mjs` also works locally, but it binds `127.0.0.1` and
serves the repo root, so it cannot be the sandbox entry point.

## Verifying in the preview

The whole game is one page (`/spiresofazeroth/`); state lives in localStorage, so a fresh page
load starts at the title screen ("Choose class" starts a run, "Enter World" resumes a saved one).
A driveable QA path: Choose class → pick a class → "Begin the climb" → click a `.sx-zone` →
click a `.sx-node.reach` → in combat press the card's number (`1`–`9`) to pick it and press it
again to play (or press once, then click the beast — the click must carry real pointer
coordinates). End Turn is `.sp-wowbtn--end`. Rewards are taken by clicking the card element
itself (`.sx-cardslot .spc`), not the slot wrapper. Screens: Settings is `.sx-set`, Codex is
`.sx-codex__item`.

## Checks

```bash
docker run --rm -v "$PWD":/repo -w /repo node:22 node tools/spires/validate-data.mjs   # data gate
docker run --rm -v "$PWD":/repo -w /repo node:22 node tools/spires/sim.mjs             # headless run, all classes
docker run --rm -v "$PWD":/repo -w /repo python:3.12-slim sh -c "pip -q install pytest && python -m pytest tests/test_spires_static.py"
```

Only the checks that cover a change are needed here; CI runs the full suite.
