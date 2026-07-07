# CLAUDE.md — AERO·TUNNEL

Single-file project: everything lives in `index.html` (HTML + inline CSS + inline vanilla JS, ~1,800 lines). No package.json, no build, no dependencies.

- **PROJECT.md** — architecture, data flow, solver physics, design decisions. Read it before touching the solver or geometry code.
- **GAPS.md** — known bugs, tech debt, and traps, ordered by severity, each with a scoped fix. Also lists things that *look* like bugs but are deliberate — check it before "fixing" anything.
- A workflow skill exists at `/mnt/skills/user/antigravity-protocol/SKILL.md` (chunk edits, plan-before-big-changes). Follow it for edit mechanics; it does not override anything here.

## Commands

- **Run:** open `index.html` in a browser, or `python3 -m http.server 8000` and visit `http://localhost:8000`. (Analytics fails silently on `file://` — expected.)
- **Build / test / lint / deploy:** none exist. "Testing" = open the page, watch the FPS counter, exercise the changed control, confirm no "flow reset" hint appears at default settings. Deploy = copy the file to any static host.
- **Syntax check after edits:** `node --check <(sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d')` or just reload the page and check the console.

## Code map (find code by section banner, not line number)

`/* ===== LBM CORE ===== */` solver arrays, constants, collide/stream/boundaries · `/* ===== TEST BODIES & MULTI-ELEMENT SCENE ===== */` geometry, DRS, rebuildBody · `/* ===== SMOKE ===== */` tracers · `/* ===== RENDERING ===== */` paintField/paintSmoke/drawBody · `/* ===== CHARTS & HISTORY ===== */` · `/* ===== STATUS ===== */` regime heuristics · `/* ===== MAIN LOOP ===== */` frame() · `/* ===== CONTROLS ===== */` all event listeners · `/* ===== EXPORT ===== */` · `/* ===== POINTER INPUT / KEYBOARD / START ===== */`.

## Conventions this file actually follows

- **State:** module-level globals under `'use strict'`. Solver state in flat `Float32Array`s indexed `i = x + y*xdim`. UI state in plain `let` variables (`bodyType`, `targetAoa`, `viewMode`, ...). No classes, no modules.
- **Units:** solver runs in **lattice units**; UI shows **m/s**. `u0` (lattice) is the solver's truth, `velocityMS` the display's. Convert only via `msToLu()`. Never feed m/s into solver math.
- **Naming:** camelCase; trailing underscore only to dodge collisions (`nN_`, `dS_`, `py_`). Cached DOM refs named `<id>El` or bare (`clv`, `hint`).
- **DOM:** elements fetched once near their section and cached. Chip groups toggle `.on` class + `aria-pressed` (boilerplate is duplicated per group — match the local pattern of the group you're editing).
- **Resets:** any change of body / speed / viscosity ⇒ call `resetGraphsAndAverages()`. AoA changes deliberately do NOT reset graphs (the polar trace accumulates across a pitch sweep).
- **Styling:** CSS custom props in `:root` (`--ink`, `--cyan`, `--amber`, ...). IBM Plex Mono for data/labels, Space Grotesk for headings. Match these; don't introduce new colors.

## Gotchas

- **Canvas ≠ grid.** Solver is 260×110; canvases are 1040×440 × DPR (capped 2). Convert with `sx = W/xdim, sy = H/ydim`; pointer input converts client → lattice coords.
- **The viscosity slider is a request.** `recomputeVisc()` may floor ν upward from the `NUF_T` table based on u0 and |α|. If forces look wrong at high α, check whether the floor changed, not the slider.
- **`rebuildBody(keepFlow, isPitching)`:** `keepFlow` is dead (ignored). `isPitching=true` skips `clearForces()` — pass it for any per-frame geometry motion or you'll zero the force averages every frame.
- **In-place streaming direction order matters.** The four loops in `stream()` iterate in specific directions so populations aren't overwritten before being read. Do not "clean up" loop order.
- **`setEquil`/`collide` clamps are load-bearing.** `U_CAP`, `RHO_MIN/MAX`, NaN checks, and the downstream density probe in `frame()` are the crash-prevention system. The probe resets the whole fluid — if you see "flow reset" during dev, your change destabilized the solver.
- **Vorticity view is pre-smoothed on purpose** (`smoothVelocity(2)`, gain 15/u0). Raising the gain or removing smoothing reintroduces a documented criss-cross artifact. Same for the edge `spongeF` layers — they kill acoustic reflections; don't remove.
- **F1 wing mode locks α to 4°** and disables α controls; DRS animates on its own fixed 0.4 s timer (`DRS_TRANSITION_TIME`), not the AOA-transition slider.
- **`nacaParams()` returns `undefined` for `'cyl'`/`'none'`** — callers early-return first by convention. New body types must handle this before calling `isPointInFoil`/`bodyOutline`.
- **`exportSnapshot()` uses a local `ox`** that shadows the particle array `ox`. Grep carefully.
- **Two identically-shaped α code paths** (slider `input` vs number-input `change` vs pitch buttons vs arrow keys) must stay in sync manually; the pitch buttons currently skip `recomputeVisc()` (GAPS #9).

## Rules

- **Never change without care:** `collide()`, `stream()`, `setEquil()`, the D2Q9 weights (`four9/one9/one36`), the streaming loop directions, the clamp constants (`U_CAP`, `WALL_U_CAP`, `RHO_*`), the sponge setup, `nuFloorRaw`/`NUF_T`, and the EMA constants (0.95 for HUD, 0.99 for polar). Each is tuned; each failure mode is a visible explosion or subtly wrong forces.
- **Keep it one file.** No bundlers, no npm deps, no external JS besides the existing GoatCounter tag. Fonts stay as the existing `@import`.
- **Preserve the reset contract:** geometry/regime changes call `resetGraphsAndAverages()`; α changes don't.
- **New controls follow the chip/slider patterns** already in `/* ===== CONTROLS ===== */`, including `aria-pressed` and keyboard reachability.
- **Edits:** use targeted string replacement (the file is large; never rewrite it wholesale). Unique anchors: use the section banners plus nearby distinctive strings, since many small expressions repeat.
- **No generated files** exist; everything is hand-written.
