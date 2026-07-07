# GAPS.md — Honest audit of AERO·TUNNEL

All paths refer to the single source file, `index.html`. Section markers like `/* ===== LBM CORE ===== */` are the reliable way to locate code; line numbers will drift.

Ordered by severity, most important first.

---

## 1. Wind-speed slider can exceed the solver's stable band (stability, HIGH)

**What:** `msToLu()` clamps to `U_MAX = 0.33` lu, and the speed slider goes to 99 m/s (≈0.33 lu). The footer and comments state the stable band is **0.02–0.20 lu**, `U_CAP` (per-cell speed clamp) is 0.28, and the viscosity-floor table `NUF_U` only covers u0 up to **0.15** — above that, the floor is extrapolated flat. Between ~60 and 99 m/s the solver is running outside every documented and tabulated safety envelope, relying solely on the NaN-probe auto-reset in `frame()`.
**Where:** constants block under `/* ===== LBM CORE ===== */` (`U_MAX`, `U_CAP`, `NUF_U`, `NUF_T`); slider `#spd` (`max="99"`).
**Why it matters:** Users at high speed + high α get repeated "flow reset" messages and garbage forces; worse, forces just below the blow-up threshold are silently unphysical.
**Fix (single task):** Either lower the slider max to 60 and set `U_MAX = 0.20`, or extend `NUF_U`/`NUF_T` with tuned rows for 0.20 and 0.33 and raise floors accordingly. The first option is a 2-line change and matches the documented band.

## 2. Moving-wall correction is applied with one global pivot and only for main-wing pitch (physics correctness, HIGH)

**What:** `currentOmega` is only set when the *main* α is easing. Flap deflection, slat deflection, and DRS motion all move geometry every frame but inject **zero** wall velocity — their motion is "teleporting geometry," exactly what the moving-wall code was added to prevent. Additionally, the wall velocity in `stream()` is computed about the global `(cx, cy)` pivot for **all** solid cells, including flap/slat/DRS elements whose actual instantaneous motion is about their own tracks/hinges.
**Where:** `updateKinematics()` (only the AoA branch sets `currentOmega`); `stream()` (`uwx = -w_omega*(y - cy); uwy = w_omega*(x - cx)`).
**Why it matters:** Transient forces during flap/slat/DRS actuation are wrong in sign/magnitude; during main pitch, forces on secondary elements get a slightly wrong wall velocity. For an educational tool this mostly shows up as odd force spikes in the time-history chart during DRS actuation.
**Fix (single task, scoped):** Minimum viable: set `currentOmega`-equivalent wall velocities per-foil. Store per-cell wall velocity in two Float32Arrays populated by `rebuildBody()` from each foil's known angular rate and pivot, and read those in `stream()` instead of the global formula. If that's too big, at least document the limitation in a comment and zero the momentum-exchange force accumulation while `drsMoving` is true so the charts don't record the artifact.

## 3. Full-grid body rebuild every frame during any transition (performance, HIGH)

**What:** While anything is easing (α, flap, slat, DRS), `updateKinematics()` calls `rebuildBody()` every frame. That's a full 260×110 double loop calling `isPointInFoil()` per foil per cell (with `cos/sin/sqrt` inside), plus a full surface-cell rescan and freed-cell refill — every 16 ms, on top of 6–12 solver steps.
**Where:** `updateKinematics()` → `rebuildBody()` → `isPointInFoil()` / `isSurfaceCell()`.
**Why it matters:** This is the dominant cause of frame drops during pitch on modest hardware; the adaptive `stepsPerFrame` then throttles the *physics* to pay for geometry rasterization.
**Fix (single task):** Restrict rasterization to each foil's bounding box (computed from `cx, cy, chord`, padded by chord×0.2), clearing only the union of old+new bounding boxes instead of `bar.fill(0)` over the whole grid. `isSurfaceCell` rescan can be limited to the same region with `barList` rebuilt from a dirty-region merge.

## 4. Zero tests, zero tooling (process, HIGH)

**What:** No test of any kind exists. No lint, no formatter, no CI. The most safety-critical pure functions — `msToLu`/`luToMs` round-trip, `nuFloorRaw` interpolation, `camber`/`thickness` NACA math, `nacaParams` custom-code parsing, `estimateStrouhal`, CSV escaping in `exportLiftCurveCsv` — are all trivially unit-testable but untested.
**Where:** everywhere; the functions listed are all under `/* ===== LBM CORE ===== */`, `/* ===== TEST BODIES ===== */`, `/* ===== EXPORT ===== */`.
**Why it matters:** Any regression in these silently corrupts physics or exports; there is no safety net for a smaller model making edits.
**Fix (single task):** Extract nothing; instead add a standalone `tests.html` (or Node script that `eval`s the extracted pure functions) asserting: `msToLu(30)===0.10`, clamping at 6 and 99 m/s; `nuFloorRaw` at all table corners; `thickness(1.0, 0.12)` ≈ trailing-edge value; NACA "2412" parses to {m:0.02, p:0.4, t:0.12}; CSV cells with quotes are escaped. Even this tiny harness catches most foreseeable regressions.

## 5. Vorticity colorbar labels no longer match the display gain (correctness of readout, MEDIUM)

**What:** The vorticity display gain was deliberately lowered from `26/u0` to `15/u0` (documented in a comment), but `drawColorbar()` still labels the extremes as `±(chord/30).toFixed(1)` — a formula that was at best loosely tied to the old gain and is now definitively wrong. The pressure label `±1.8` also disagrees with the actual clamp in `paintField()` (`cp*0.55` saturates at |cp| ≈ 1.82 — coincidentally close, but the speed view's "0.85/1.7" midpoint/max are only right because `ss = 1/(u0*1.7)`).
**Where:** `paintField()` (gain `vs = 15/u0`), `drawColorbar()` (label strings).
**Why it matters:** This is a tool people use to *learn* CFD; a quantitatively wrong colorbar is worse than an unlabeled one.
**Fix (single task):** Compute labels from the actual gains: vorticity max = `1/vs` scaled by whatever normalization the `ω·c/u0` unit implies (`(1/15)·chord` in these units → derive and label from the constant), or simplify the labels to `min/0/max` with "(qualitative)".

## 6. Dead code and dead parameters (tech debt, MEDIUM)

**What & where:**
- `rebuildBody(keepFlow, isPitching)` — `keepFlow` is never read; all call sites pass a value, implying behavior that doesn't exist.
- `luToMs()` is defined and never called.
- In `drawBody()`, a `grad` linear gradient is created before the F1 branch and immediately shadowed by a second identical `grad` inside the `else` — the first is dead work every frame.
- The comment in `rebuildBody()` ("only solidifying if they don't clip the main structure") describes masking logic that the code doesn't do — the loop just `break`s on first containment; main wing wins only via the earlier `continue`.
- `.prow` has `transition: all .3s ease` but `.hidden` toggles `display:none`, which never animates.
**Why it matters:** Each of these misleads a future editor about what the code does; `keepFlow` in particular invites someone to "fix" call sites.
**Fix (single task):** Remove `keepFlow` from the signature and all 8+ call sites; delete `luToMs`; delete the outer `grad`; rewrite the misleading comment; drop the no-op transition.

## 7. High-speed CSV/Newton outputs stamp *current* solver settings onto historical rows (data integrity, MEDIUM)

**What:** `exportLiftCurveCsv()` writes `bodyName()`, `u0`, and `nu` — evaluated at export time — onto **every** historical row of `pHist`. This is currently mostly safe because body/speed/viscosity changes call `resetGraphsAndAverages()`, but two writes break the invariant: (a) `nu` changes automatically via `recomputeVisc()` whenever α crosses floor-table thresholds *without* clearing `pHist`, so a pitch sweep exports rows whose true ν varied but are all stamped with the final ν; (b) `bodyName()` for F1 includes live DRS state, which can change without a graph reset.
**Where:** `/* ===== EXPORT ===== */`, `exportLiftCurveCsv()`; `recomputeVisc()`.
**Why it matters:** The CSV is the tool's only quantitative export; a student comparing CL vs α at "constant ν" is being lied to at high α.
**Fix (single task):** Store `nu` (and DRS state for F1) into each `pHist` entry at `pushPolar()` time and export the per-row values.

## 8. Third-party analytics script with no integrity control (security, LOW-MEDIUM)

**What:** `<script async src="//gc.zgo.at/count.js">` loads remote code with no SRI hash, no CSP on the page, protocol-relative URL. If gc.zgo.at is compromised or the page is served over HTTP, arbitrary JS runs in the page. The protocol-relative URL also silently fails under `file://`.
**Where:** last script tag in `<body>`.
**Why it matters:** Low stakes (no user data, no auth), but it is the only remote-code trust decision in an otherwise self-contained file, and it undermines the "portable single file" story.
**Fix (single task):** Change to `https://gc.zgo.at/count.js`, add `crossorigin="anonymous"` + an SRI `integrity` hash, or drop analytics for offline distribution.

## 9. Blocking `alert()` for AoA validation; inconsistent validation patterns (UX/consistency, LOW)

**What:** The numeric α input uses a modal `alert()` on out-of-range values, then clamps. Every other control clamps silently (`updateSpeedFromMs`, NACA input just resets its placeholder). Also, `aoaInput` `change` calls `recomputeVisc()` but the arrow-key path routes through the slider's `input` handler — two parallel code paths for "α changed" that must be kept in sync manually (a third exists in the PITCH ±8° buttons, which update `targetAoa` and both inputs by hand and *don't* call `recomputeVisc()` — the floor only updates once easing starts changing... actually it never recomputes until another control fires; this is a real, if minor, bug: pitch-button sweeps to high α run on the old viscosity floor until the next unrelated `recomputeVisc()` call).
**Where:** `aoaInput` `change` listener; `#btnPitchMinus`/`#btnPitchPlus` listeners; keyboard `ArrowLeft/Right` handler.
**Why it matters:** The recompute gap means the stability floor lags target α during button-driven sweeps — mildly increases blow-up risk at exactly the moments users are stress-testing stall.
**Fix (single task):** Create one `setTargetAoa(value)` helper that clamps, syncs slider + number input, and calls `recomputeVisc()`; route all four entry points (slider, number input, buttons, keyboard) through it. Replace `alert()` with the silent-clamp pattern.

## 10. Repeated chip-group boilerplate (tech debt, LOW)

**What:** The pattern "querySelectorAll the group, remove `.on`, add `.on` to clicked, sync `aria-pressed`" is hand-duplicated in at least seven handlers (`#bodies`, `#configs`, `#fluids`, `#uPresets`, `#quality`, view buttons, plus `syncPressed` covering only some of them). Several duplicates drift: `#quality` sets `aria-pressed` via `syncPressed`, `#uPresets` does it inline, view buttons do it inline with a different idiom.
**Where:** `/* ===== CONTROLS ===== */`.
**Why it matters:** Every new chip group re-invites the drift; a11y state is already inconsistent (the APPLY button carries a meaningless `aria-pressed`).
**Fix (single task):** Write `selectChip(groupSelector, chipEl)` that does classes + aria in one place; refactor all groups to use it.

## 11. Fragile/undefined behavior on unexpected geometry inputs (edge cases, LOW)

**What:**
- `nacaParams()` returns `undefined` for `'cyl'`/`'none'`; safe only because callers happen to early-return first. Adding a new body type and forgetting one early-return produces a TypeError deep in `isPointInFoil`.
- `bodyOutline()` would crash for a cylinder foil descriptor; safe only by the same convention.
- Custom NACA codes like `0099` produce t clamped to 0.30 but codes like `9012` produce m=0.09 with p from digit 2 forced to ≥0.1 — extreme-camber shapes can rasterize to disconnected cell blobs at this grid resolution, and `barList` force sums over such blobs are meaningless. No guard exists.
**Where:** `nacaParams`, `isPointInFoil`, `bodyOutline`, `applyCustomNaca`.
**Why it matters:** Extending the body roster is the most likely future change; the implicit conventions are traps.
**Fix (single task):** Make `nacaParams` throw (or return a default) with a clear message for unknown types, and add a post-rasterization sanity check in `rebuildBody` (e.g., `barList.length < 8` for a non-`none` body ⇒ show hint "body too thin for grid").

## 12. Mobile layout likely illegible (UX, LOW)

**What:** The ≤700 px media query sizes HUD text in `cqi` units as small as `1cqi` — on a 380 px container that is ~3.8 px text. `container-type:inline-size` is only set inside this query, and several `!important` overrides fight the base styles.
**Where:** `@media (max-width:700px)` block in the `<style>`.
**Why it matters:** The mobile HUD is effectively invisible; the hint line likewise.
**Fix (single task):** Replace the sub-2cqi font sizes with `max(9px, Ncqi)` via `font-size: max(9px, 1.5cqi)` and verify on a 380 px viewport.

## 13. Minor inconsistencies and half-finished polish (LOW)

- **Naming:** `nN_` / `dS_` trailing underscores vs. every other clean name; `py_`/`px_` shadow-avoidance underscores in render code; `ox/oy` particle arrays vs. `ox` local canvas context in `exportSnapshot()` (same identifier, different meanings — genuinely confusing when grepping).
- **Copy/typos:** footer "Feedback is appreciated!," (stray comma), "hosted in my github account" with no actual link; the About card has an empty `<span class="handle"></span>`.
- **`syncPressed('#bodies .chip')`** runs over the APPLY button too, giving it `aria-pressed` semantics it shouldn't have.
- **`updateKinematics` reads `#pitchTime` via `getElementById` every frame** instead of the cached `pitchTimeEl` that already exists 200 lines below — duplicate lookup and inconsistent with the caching convention used everywhere else.
- **DRS "Macarena Wing Style" label** overflows its chip on narrow screens.
**Fix (single task each):** rename `exportSnapshot`'s context to `octx`; use the cached `pitchTimeEl`; fix copy; scope `syncPressed` selectors to `[data-b]`.

---

## Not gaps, but easy to mistake for gaps (do not "fix")

- Graphs deliberately survive pitch transitions (`clearHist/clearPolar` intentionally removed from `rebuildBody` — comment says so).
- The viscosity floor overriding the slider is a feature, not a bug.
- The vorticity pre-smoothing and lowered gain are deliberate artifact suppression — reverting them reintroduces a criss-cross rendering artifact documented in comments.
- The sponge layers exist to absorb acoustic reflections; removing them brings back standing-wave patterns.
