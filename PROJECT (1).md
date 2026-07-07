# PROJECT.md — AERO·TUNNEL

## What this is

AERO·TUNNEL is a real-time, interactive 2D wind tunnel that runs entirely in the browser. A user picks a test body (NACA airfoils, a flat plate, a cylinder, a two-element F1 rear wing, or an empty tunnel), sets angle of attack, wind speed, and fluid viscosity, and watches the flow develop live: vorticity/speed/pressure fields, smoke tracer streaks, lift and drag readouts, a lift-curve plot, and a force time history. It also supports multi-element high-lift configurations (leading-edge slat, trailing-edge Fowler flap), animated pitch transitions with a moving-wall boundary correction, and an F1 DRS flap with two opening kinematics (normal trailing-edge pivot, and a 180°-flip "Macarena wing" mode).

Audience: it's an educational/demo tool by Sarujan Srikaran, aimed at people learning aerodynamics or CFD, and at anyone who wants a visually rich, zero-install flow toy. It is explicitly *not* a validated CFD code — the footer says so, and the physics section below explains the compromises.

## Tech stack and why

| Piece | Choice | Evident reasoning |
|---|---|---|
| Everything | One `index.html` file: inline CSS, inline vanilla JS | Portability. A comment says fonts/styles live in-file "so the demo stays portable." No build step, no bundler, works from a static host or `file://` (mostly — see GAPS on the analytics script). |
| Solver | Hand-rolled D2Q9 Lattice-Boltzmann (BGK) in plain JS with `Float32Array`s | LBM is uniquely suited to real-time browser CFD: local operations, no linear solves, trivially handles arbitrary solid masks. Typed arrays in flat structure-of-arrays layout keep the hot loops JIT-friendly. |
| Rendering | Three stacked 2D `<canvas>` layers + offscreen `ImageData` buffer | The scalar field is painted at solver resolution (260×110) into a small canvas, then upscaled with `imageSmoothingEnabled` — cheap and smooth. Layering (field / smoke / body) lets each layer update at its own cadence and composite independently. No WebGL, presumably for simplicity and compatibility. |
| UI | Hand-written HTML controls, no framework | The control surface is small enough that a framework would only add weight. State lives in module-level JS variables. |
| Fonts | Google Fonts (`Space Grotesk`, `IBM Plex Mono`) via `@import` | Aesthetic only. |
| Analytics | GoatCounter (`gc.zgo.at`) | Lightweight, privacy-friendly page counting. Only external script. |

There are **no dependencies, no package.json, no tests, no build, no lint config**. The entire project is this one file.

## Architecture

```
┌────────────────────────────── index.html ──────────────────────────────┐
│                                                                        │
│  UI controls (chips/sliders) ──► global state (bodyType, targetAoa,    │
│        │                          velocityMS→u0, nuSlider, configMode, │
│        │                          drsOpen, viewMode, NP, ...)          │
│        ▼                                                               │
│  rebuildBody() ── rasterizes getSceneFoils() into `bar` solid mask     │
│        │          + `barList` surface cells; refills freed cells       │
│        ▼                                                               │
│  frame() rAF loop (adaptive stepsPerFrame 4–14):                       │
│    updateKinematics()   — eases aoa/flap/slat/DRS toward targets,      │
│                           sets currentOmega, rebuilds body per frame   │
│    ×N: setBoundaries() → collide() → stream() → advectParticles()      │
│         │inlet Dirichlet   │BGK +        │pull-streaming +             │
│         │slip walls        │Smagorinsky  │bounce-back w/ moving wall   │
│         │                  │+ sponge τ   │+ momentum-exchange forces   │
│    force accumulation (Fx,Fy) ─► CL/CD ─► smoothing (smCl 0.95 EMA,    │
│                                   longSm 0.99 EMA) ─► HUD, charts,     │
│                                   real-Newton estimate (ρ=1.204,       │
│                                   1 m chord, displayed m/s)            │
│    paintField() (jet colormap; vorticity view pre-smooths velocity)    │
│    paintSmoke() (fading streak trails)                                 │
│    drawBody()/drawPolar()/drawTime()/drawColorbar()                    │
│                                                                        │
│  Exports: PNG snapshot (composites all layers), CSV of polar history   │
│  Pointer: click/drag paints smoke puffs   Keyboard: full shortcut set  │
└────────────────────────────────────────────────────────────────────────┘
```

### The solver (the load-bearing core)

- **Grid**: `xdim=260, ydim=110` lattice cells. Display canvases are 1040×440 CSS px × DPR (capped at 2). Everything physical happens on the small grid; the big canvases are pure presentation.
- **Data layout**: nine `Float32Array(N)` distributions (`n0, nN_, nS, nE, nW, nNE, nSE, nNW, nSW`), plus `rho, ux, uy`, a `Uint8Array bar` solid mask, and `barList` (flat `[x,y,...]` list of *surface* solid cells only).
- **Units**: the solver runs in lattice units. The UI shows m/s; `msToLu()` maps 30 m/s → 0.10 lu linearly, clamped to `[U_MIN=0.02, U_MAX=0.33]`. `U_REF=0.10`, `MS_REF=30`. The documented "stable band" is 0.02–0.20 lu (footer), which the slider can exceed — see GAPS.
- **Collision**: BGK with a Smagorinsky-style eddy-viscosity correction (`Cs2=0.03`): the non-equilibrium stress magnitude `Q` locally raises the effective relaxation time `tauEff`, stabilizing under-resolved turbulent regions.
- **Sponge layers**: `spongeF` adds extra relaxation time near all four domain edges (strongest at the outlet) to absorb acoustic reflections — a long comment in the file explains this was added to kill a criss-cross standing-wave artifact. Do not remove it.
- **Boundaries**: hard-equilibrium inlet column at `u0`; free-slip top/bottom (population mirroring); zero-gradient outlet (copy second-last column).
- **Moving wall / dynamic pitch**: during an AoA transition, `currentOmega` (rad per solver step, about global pivot `cx,cy`) injects wall velocity into the bounce-back rule via momentum-exchange correction terms (`d9_*`, `d36_*`), capped at `WALL_U_CAP=0.08`. This is what makes pitch transitions physically stir the fluid instead of teleporting geometry.
- **Forces**: momentum exchange summed over `barList` during `stream()`, accumulated into `Fx, Fy`, averaged over `forceSamples`, normalized by `q = 0.5·u0²·Lref` (Lref = chord, or cylinder diameter). Real Newtons are then derived from the *displayed* m/s with ρ=1.204 kg/m³ and an assumed 1 m × 1 m wing.
- **Stability failsafes** (all deliberate, all load-bearing):
  - `setEquil` and `collide` clamp density to `[RHO_MIN=0.35, RHO_MAX=3.0]` and speed to `U_CAP=0.28`; non-finite cells are reset to inlet equilibrium.
  - `recomputeVisc()` enforces a **viscosity floor** from a bilinearly-interpolated table `NUF_T` (indexed by u0 and |AoA|), scaled ×1.2 for bluff bodies. The slider viscosity is a *request*; the floor can override it (UI shows `*` and "(floored)").
  - A density probe downstream of the body triggers a full `initFluid()` + graph reset if it goes non-finite/out-of-range ("flow reset — try higher viscosity or lower speed").

### Geometry system

`getSceneFoils()` returns a list of foil descriptors `{type, cx, cy, chord, a, isMain}` depending on `bodyType` and `configMode`:

- 4-digit NACA sections are generated analytically (`camber()`, `thickness()`, standard NACA polynomials, thickness floor 0.012).
- Multi-element: TE flap follows a curved Fowler track (translate aft + drop, then deflect), LE slat follows an arc track around the nose. Both are `n4415` sections scaled down.
- F1 wing: a thick negative-camber `f1main` (m=−0.10, pitched −20° relative to the locked α=4°) plus an `f1flap` DRS element that rotates about either its trailing edge (normal DRS) or its mid-chord (180°-flip mode, opening 230° from closed at −48°). DRS animates on its own fixed 0.4 s timer, independent of the AoA-transition slider.
- `rebuildBody()` rasterizes all foils into `bar`, rebuilds `barList`, and — crucially — when solid cells become fluid (geometry moved), refills them with the average of neighboring fluid state rather than raw equilibrium, so moving bodies don't inject shockwaves.

### Visualization pipeline subtleties

- **Vorticity view**: velocity is smoothed with two separable [1,2,1] passes (`smoothVelocity`) *before* taking the curl, and the display gain was deliberately lowered from 26 to 15/u0. Both changes exist to suppress grid-scale ripple that otherwise renders as a fake criss-cross net. The scalar field then gets two more smoothing passes for all views. Comments document this history — treat these numbers as tuned, not arbitrary.
- Solid cells are painted as the average of neighboring fluid values (in-painting) so the colormap doesn't show a hard hole under the drawn body.
- Smoke is up to 6000 massless tracers (`NP` active), advected with a midpoint (RK2-style) step through bilinearly interpolated velocity, drawn as short streaks with `destination-out` fading. Particles respawn at inlet lanes, on leaving the domain, or on hitting the body. Streaks longer than 6 lu are skipped (respawn teleport suppression).

### Charts and status

- Time history: 600-sample ring buffer of lightly-smoothed CL/CD (`smCl/smCd`, 0.95 EMA), pushed every solver batch.
- Lift curve ("polar"): persistent trace of heavily-smoothed (`longSmCl/longSmCd`, 0.99 EMA) CL and CD×2 vs α, capacity 3000 points, pushed every 3rd frame. Survives pitch transitions on purpose — a comment notes `clearHist/clearPolar` were intentionally removed from `rebuildBody` so pitching sweeps trace out the lift curve.
- `resetGraphsAndAverages()` is the one sanctioned way to wipe everything (called on body/speed/viscosity change, manual reset, and NaN recovery).
- `flowRegime()` is a *heuristic* label (STABLE/PRE-STALL/STALLING/POST-STALL/VORTEX SHEDDING) from |α| plus CL fluctuation amplitude — it is not measured separation.
- `estimateStrouhal()` counts mean-crossings of CL for the cylinder only.

## Key design decisions (inferred)

1. **Single file, zero build** — portability trumps everything. Every extension so far respects this.
2. **UI in physical units, solver in lattice units** — the m/s slider was clearly retrofitted (comments reference the mapping repeatedly); `u0` is the single source of truth for the solver, `velocityMS` for display/Newtons.
3. **Stability over accuracy** — viscosity floors, velocity caps, density clamps, sponges, and NaN auto-recovery all sacrifice fidelity so the demo never visibly explodes. Numbers like `Cs2=0.03`, sponge strengths, and the `NUF_T` table are empirically tuned.
4. **Graphs persist through pitch, reset on regime change** — deliberate, documented in comments; enables the "sweep α and watch the lift curve draw itself" workflow.
5. **Presentation decoupled from physics** — render modes, smoke, DPR scaling, and smoothing never touch solver arrays (except read-only), so display changes are always safe.
6. **Adaptive workload** — `stepsPerFrame` self-tunes between quality-preset bounds from frame time; physics rate therefore varies with hardware.

## Critical paths (rank-ordered)

1. **`collide()` / `stream()` / `setEquil()`** — the numerical core. Any change here risks instability or wrong forces. The population index conventions (e.g., `nN_` with trailing underscore to avoid a name clash) and the streaming loop *directions* (which make in-place streaming correct) are extremely easy to break.
2. **`rebuildBody()` + `getSceneFoils()` + `isPointInFoil()`** — geometry-to-mask. Runs every frame during any animated transition (a known perf hotspot), and the freed-cell refill logic prevents visual shockwaves.
3. **`recomputeVisc()` / `nuFloorRaw()` / `msToLu()`** — the stability envelope. Changing clamps or the floor table can make previously-safe UI states explode.
4. **`frame()`** — orchestration, force averaging, EMA smoothing constants, NaN probe. The EMA constants (0.95, 0.99) define the visual character of every readout.
5. Safe to change casually: CSS, HUD text, card copy, colorbar labels, chart cosmetics, export formatting, keyboard mappings, smoke visual style.

## Surprises / gotchas for newcomers

- **The display canvas is 4× the solver grid.** Coordinates in pointer handlers are converted to lattice space; anything drawn on `bodyCx` uses `sx = W/xdim, sy = H/ydim` scale factors.
- **`nN_` has a trailing underscore** (and `dS_` inside `collide`) purely to dodge identifier collisions. Not a typo.
- **The viscosity slider lies to you** at high speed/AoA: the effective ν is floored, and only the small `*` and "(floored)" tag reveal it. Re shown in the HUD uses the *effective* ν.
- **`rebuildBody(keepFlow, isPitching)` — `keepFlow` is dead**; every caller passes it but nothing reads it. `isPitching=true` is what prevents force-accumulator resets mid-transition.
- **Changing α does NOT reset graphs; changing body/speed/viscosity DOES.** Intentional.
- **Wind speeds above ~66 m/s exceed the documented 0.20 lu stable band** (clamp is 0.33); the NaN-recovery probe is the safety net up there.
- **The F1 wing locks α to 4°** and disables the α controls and AoA-transition row; DRS uses its own hardcoded 0.4 s easing.
- **`drawBody()` masks overlapping element outlines** with `destination-out` against the main wing so the assembly reads as one machined part — the physics mask does no such subtraction (main wing wins by early `continue`).
- **GoatCounter analytics** is loaded protocol-relative (`//gc.zgo.at`) — fails silently on `file://`, and is an external script trust decision.
