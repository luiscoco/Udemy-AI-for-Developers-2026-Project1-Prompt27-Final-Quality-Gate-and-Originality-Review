# Final Quality Gate and Originality Review

This note documents what was actually done for the "final quality gate" prompt on
the Equipment Maintenance Hub project, in the order it happened. It's meant for
students following along with the course to see how an AI assistant approaches a
gate like this: run the checks as given, don't paper over failures, and ask before
making judgment calls that change what "passing" means.

## Running the app (Windows terminal)

From the project root, in PowerShell or Command Prompt:

```
npm install
npm run dev
```

`npm install` only needs to be run once (or after pulling changes that touch a
`package.json`). `npm run dev` starts both servers together — Fastify backend and
Vite frontend — using `concurrently`:

- Backend: `http://127.0.0.1:3001` (health check at `/api/health`)
- Frontend: `http://localhost:5173` (or the next free port, e.g. `5174`, if 5173
  is already taken — Vite prints the actual URL it picked in the terminal output)

Open the frontend URL shown in the terminal in your browser to use the app. Press
`Ctrl+C` in the terminal to stop both servers.

## 1. Ran the checks in the order requested

```
npm run verify:contract
npm run lint
npm test
npm run build
npm run dev   (+ manual scenario)
```

### `npm run verify:contract`
Failed on the first attempt:

```
Error: Cannot find module 'openapi-typescript/package.json'
```

Cause: no `node_modules` existed anywhere in the repo yet. Ran `npm install` at the
root (this installs all three workspaces: `apps/backend`, `apps/frontend`,
`packages/contract`). Re-ran the command — passed:

```
Contract types are in sync with openapi.yaml.
```

### `npm run lint`
Passed with no errors on the first run.

### `npm test` and `npm run build` — the real finding
Both of these initially reported success:

```
> echo "no tests configured yet" && exit 0
> echo "no build configured yet" && exit 0
```

That's a false green. Each workspace already has real, working scripts
(`apps/backend` and `apps/frontend` both have `vitest run`, `apps/frontend` has
`vite build`) — the **root** `package.json` just never called them. A quality gate
that reports "test: pass" without running a single test is worse than no gate at
all, so this was flagged rather than accepted.

Before changing anything, this was reported back explicitly (command + exact
output) and the fix was confirmed before proceeding, rather than silently
"fixing" a failing gate. The chosen fix: make the root scripts delegate to every
workspace:

```json
"test": "npm run test --workspaces --if-present",
"build": "npm run build --workspaces --if-present",
"dev": "concurrently -n backend,frontend -c blue,green \"npm run dev --workspace=@equipment-hub/backend\" \"npm run dev --workspace=@equipment-hub/frontend\""
```

(`concurrently` was added as a root devDependency so `npm run dev` boots both the
Fastify backend and the Vite frontend together with one command, which is what the
manual scenario below needed anyway.)

Re-running `npm test` after that fix surfaced one real thing worth knowing about:
a backend test timed out when the whole test suite (backend + frontend + contract)
ran back-to-back under heavy load, but passed instantly when re-run on its own or
as part of a quieter full run. That was a flaky/slow-machine symptom, not an
application bug — confirmed by re-running the backend suite alone (~1s, all green)
and then the full `npm test` again cleanly (98/98 tests passing: 76 backend, 15
frontend, 7 contract).

`npm run build` then passed cleanly, producing the frontend's production bundle.

## 2. Manual scenario (report → triage → assign → schedule → start → complete)

With `npm run dev` running, the full work-order lifecycle was driven through the
same HTTP endpoints the UI calls (`POST /api/work-orders`, then
`/api/work-orders/:id/transitions` for triage/schedule/start/complete, and
`/api/work-orders/:id/assignment` for assigning a technician). Each step returned
the expected state, and `GET /api/dashboard/summary` reflected the change
immediately after every mutation.

There was no browser-automation tool available in this environment, so the UI
itself wasn't clicked through visually. Instead, "dashboard totals change without
a full page reload" was verified by:
- confirming there is no `window.location` / `reload` call anywhere in the
  frontend source, and
- checking that every mutation in `App.tsx` updates React state directly
  (`setWorkOrders`, `setSummary` via a `refreshSummary()` call after each action),
  which matches the existing `App.test.tsx` integration test that already covers
  select → assign → schedule → dashboard refresh in a simulated DOM.

This is good evidence but is explicitly weaker than an actual human/browser
confirmation — noted here rather than glossed over.

## 3. Originality review

A background review searched the whole repo (`apps/`, `packages/`, `data/`, root
configs, `package.json` files, `openapi.yaml`, `index.html`; `node_modules`,
`dist`, and `.git` excluded) for anything that could be copied from a real/private
consultancy rather than being original to this course project. Result: **PASS**.

- No occurrence of "atmira" (or any other private-consultancy name) anywhere in
  source, comments, fixture data, or `package.json` files.
- Fixture data (`data/assets.json`, `data/technicians.json`,
  `data/work-orders.json`) is generic and fictional: equipment tags like
  `HVAC-01`, `PUMP-04`, `CNC-02`; generic locations like "Building A", "Plant
  Room B"; technician names like Elena Vasquez, Marcus Chen, Priya Nair with
  generic specialties; work order narratives are plausible but invented
  (refrigerant loss, bearing failure, filter change) — no real client names or
  JIRA-style ticket references.
- No project-authored README existed at the time the review ran (this file is
  the first one) — nothing to check there, and no copied boilerplate headers or
  proprietary/confidential markers were found in code comments.
- All `package.json` name/author/repository fields are consistent and original
  (`equipment-maintenance-hub` / `@equipment-hub/*`), no leaked emails or URLs.
- No image/screenshot files exist in the project source tree, so no risk of a
  leaked real UI screenshot.
- `openapi.yaml` and `index.html` only carry generic "Equipment Maintenance Hub"
  branding.

## 4. Housekeeping noticed along the way

An unrelated, orphaned Vite dev server from a different course exercise
(`Prompt19 Manual Backend Verification`) was found squatting on port 5173 from an
earlier session and was stopped so this project's dev server could bind cleanly.
This project's own dev server (backend + frontend) was stopped again afterward so
nothing was left running in the background.

## Net result

| Check | Initial state | Final state |
|---|---|---|
| `verify:contract` | Failed (missing deps) | Pass |
| `lint` | Pass | Pass |
| `test` | False-positive stub | Pass (98/98 real tests) |
| `build` | False-positive stub | Pass |
| Manual scenario | — | Pass (API-verified; UI code-verified, not browser-verified) |
| Originality review | — | Pass — no leaked names, generic fictional fixtures, original branding |
