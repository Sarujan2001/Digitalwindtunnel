# AERO·TUNNEL — Lattice-Boltzmann Wind Tunnel

AERO·TUNNEL is a browser-based, real-time educational wind tunnel built with a D2Q9 Lattice-Boltzmann Method solver. It visualises flow around airfoils, multi-element wings, a cylinder, a flat plate, and an F1-style rear wing with DRS.

The project is designed as an interactive learning tool for aerodynamics, flow visualisation, stall behaviour, vortex shedding, and basic force trends.

## Features

- Real-time 2D Lattice-Boltzmann wind tunnel
- D2Q9 LBM solver with BGK collision
- Smagorinsky-style eddy-viscosity stabilisation
- Vorticity, speed, and pressure render modes
- Smoke tracer visualisation
- Live lift coefficient, drag coefficient, L/D ratio, Reynolds number, and FPS display
- Flow regime indicator, including attached flow, pre-stall, stall, post-stall, and vortex shedding
- Built-in test bodies:
  - NACA 0012
  - NACA 2412
  - NACA 4415
  - Flat plate
  - Cylinder
  - Empty tunnel
  - F1 rear wing with DRS
- Custom 4-digit NACA airfoil input
- Multi-element wing modes:
  - Single element
  - Leading-edge slat only
  - Trailing-edge flap
  - Slat + flap high-lift configuration
- Dynamic angle-of-attack transition
- Pitch up, pitch down, and snap controls
- Adjustable wind speed in metres per second
- Internal conversion from m/s to safe LBM lattice velocity
- Fluid viscosity presets, including air, water, olive oil, honey, and glycerin
- Live plots for:
  - Lift curve
  - Force time history
  - Pressure coefficient distribution
- PNG snapshot export
- Lift curve CSV export
- Responsive layout for desktop and mobile

## Demo

Open `index.html` directly in a browser, or host it with GitHub Pages.

```bash
# Option 1: open directly
index.html

# Option 2: run a local server
python -m http.server 8000
```

Then open:

```text
http://localhost:8000
```

## How to Use

1. Choose a test body from the control panel.
2. Adjust the target angle of attack using the slider or number input.
3. Set the wind speed in m/s.
4. Choose a render mode:
   - Vorticity
   - Speed
   - Pressure
5. Turn smoke on or off to visualise flow paths.
6. Use the fluid preset or viscosity slider to change Reynolds-number behaviour.
7. Use the live plots to observe lift, drag, and pressure trends.
8. Export a snapshot or lift-curve CSV if needed.

## Keyboard Controls

| Key | Action |
|---|---|
| `Space` | Pause or resume simulation |
| `R` | Reset the flow |
| `S` | Toggle smoke |
| `V` | Cycle render mode |
| `[` / `]` | Decrease or increase smoke density |
| `←` / `→` | Change angle of attack |
| `Shift + ←` / `Shift + →` | Coarse angle-of-attack step |
| Click / drag | Paint extra smoke into the tunnel |

## Physics Model

The solver uses a D2Q9 Lattice-Boltzmann Method with BGK collision. The simulation runs internally in lattice units, while the user interface displays wind speed in metres per second for readability.

The wind-speed control maps real-world-looking values into a stable lattice velocity range. This keeps the simulation interactive and stable inside a browser.

The solver includes:
- equilibrium distribution functions,
- streaming and collision steps,
- bounce-back solid boundary handling,
- free-slip top and bottom boundaries,
- outlet copying for flow exit,
- sponge damping layers near domain boundaries,
- local eddy-viscosity stabilisation,
- smoke particles advected by the computed velocity field.

## Important Notes

This project is an educational visual solver, not a certified CFD tool. The displayed lift, drag, Reynolds number, pressure coefficient, and force values are useful for visual learning and relative comparison, but they should not be used as validated engineering data without proper CFD or experimental verification.

The solver is low-Mach and intended to stay within a stable LBM lattice-velocity band. Very high speeds or aggressive geometries may produce numerical artefacts.

## Project Structure

The current version is contained in a single portable HTML file:

```text
index.html
```

The file includes:
- HTML user interface
- CSS styling
- JavaScript LBM solver
- render logic
- smoke tracer system
- body generation
- plotting
- export tools

## Suggested Future Improvements

- Import custom `.dat` airfoil coordinate files
- Add mesh/grid display toggle
- Add clearer solver warning when velocity approaches unstable limits
- Add more airfoil presets
- Add validation notes comparing basic NACA lift trends with reference data
- Add mobile-specific simplified control panel
- Add separate documentation for the numerical method
- Add optional tutorial mode for students

## Built By

Built by **Sarujan Srikaran**.

LinkedIn: [Sarujan Srikaran](https://www.linkedin.com/in/sarujansrikaran)



Copyright (c) 2026 Sarujan Srikaran
```
