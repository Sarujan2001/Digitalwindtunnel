# AERO·TUNNEL

A real-time, interactive 2-D wind tunnel that runs entirely in your browser. No installs, no server, no build step — one HTML file containing a full lattice-Boltzmann fluid solver, live force measurement, and smoke visualisation.

Built by **Sarujan Srikaran** ([@SarujanSrikaran](https://github.com/Sarujan2001))

---

## What it does

- Simulates airflow over **NACA airfoils** (0012, 2412, 4415, or any custom 4-digit code), a **flat plate**, a **cylinder**, or an empty tunnel
- **Measures lift and drag** directly from the simulation using momentum exchange on the body surface — the coefficients are computed, not looked up
- Traces the **lift curve (Cl vs α) live** as you sweep the angle-of-attack slider
- Shows a **real-time flow status badge**: STABLE FLOW → PRE-STALL → STALLING → POST-STALL, driven by both the angle and the measured lift unsteadiness
- Estimates the **Strouhal number** of vortex shedding behind the cylinder
- Renders **vorticity, speed, or pressure** fields plus smoke streaklines you can paint by clicking and dragging
- Exports **PNG snapshots** and **lift-curve CSV** files

## Controls

| Action | Control |
|---|---|
| Pause / resume | `Space` |
| Reset the flow | `R` |
| Smoke on / off | `S` |
| Cycle view mode (vorticity / speed / pressure) | `V` |
| Smoke density | `[` / `]` |
| Angle of attack | `←` / `→` (hold `Shift` for coarse steps) |
| Paint smoke | Click or drag anywhere on the tunnel |

All controls are also available as on-screen sliders and buttons, with tooltips.

## The physics (short version)

The solver is a **D2Q9 lattice-Boltzmann method** with BGK collision, stabilised at higher Reynolds numbers by a Smagorinsky-type eddy-viscosity term computed from the local non-equilibrium stress. Boundaries: velocity inlet, far-field top/bottom walls, and a zero-gradient outlet behind a viscous sponge layer that absorbs pressure waves. Aerodynamic forces come from momentum exchange on surface lattice links. Smoke tracers are advected with second-order Runge-Kutta so streaklines hug curved flow.

The full mathematical write-up — every equation as implemented, aimed at readers with a CFD background — is in [`docs/aero_tunnel_methods.pdf`](docs/aero_tunnel_methods.pdf).

**Honest limits:** the flow is 2-D at model-scale Reynolds numbers in a closed test section, so absolute coefficients differ from full-scale flight. The *trends* — lift slope, stall onset, drag polar shape, Karman shedding — are the real thing.

MIT — see [LICENSE](LICENSE). Attribution appreciated.
