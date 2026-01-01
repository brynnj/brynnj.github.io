---
title: "6-DOF Rigid Body Simulator + Web Viewer"
layout: project
tags: [simulation, dynamics, python, visualization, babylonjs]
---

# 6-DOF Rigid Body Simulator + Web Viewer

## Overview
A Python-based 6-DOF rigid body simulator with a Babylon.js web viewer. It runs
multi-body dynamics with configurable force/moment models, exports structured
outputs, and provides a browser-based playback tool for inspection and demos.

## Problem
I wanted to build my own rigid body dynamics platform so I could mess around with orbital mechanics, flight dynamics, etc. without being bound to anything off-the-shelf.

## Approach / Architecture
The simulator integrates the 6-DOF ODEs for each body, aggregates forces from a
pluggable model system, and writes artifacts for analysis and visualization:
`sim_output.npz` (raw state history), `sim_output_plots.pdf` (summary plots),
and `viewer/sim_output.json` (viewer-ready animation data).

## Key technical details
- Quaternion-based orientation propagation with normalization after integration.
- Force/moment model interface (`ForceMomentModel`) with implementations for
  gravity, thrusters, and springs.
- Multi-body state packing/unpacking so all bodies share one ODE solve call.
- Postprocessing pipeline that exports both analytics (PDF) and visualization
  payloads (JSON for Babylon.js).

## Results
- Generates per-body state plots and motion traces suitable for quick QA.
- Produces a lightweight JSON animation for the web viewer with time-aligned
  position and quaternion samples.

<figure>
  <img src="/assets/img/sixdof-sim-states.png" alt="State plots for simulated bodies">
  <figcaption>
    Example state summary output (position, velocity, attitude, rates).
  </figcaption>
</figure>

<figure>
  <img src="/assets/img/sixdof-sim-viewer.png" alt="Babylon.js playback view">
  <figcaption>
    Web viewer playback of multi-body motion for quick demos and reviews.
  </figcaption>
</figure>

## How to run / reproduce
1. Install dependencies with Poetry.
2. Run `poetry run python main.py` to generate the `.npz`, `.pdf`, and viewer JSON.
3. Open `viewer/index.html`, or serve the repo and browse to `/viewer/`.

## Limitations
- No built-in validation against external dynamics benchmarks yet.
- No implementation of constraints or joints - not set up for kinematic trees, just independent bodies that can interact with arbitrary force laws.
- Force models are simple and constant; time-varying controls are minimal.
- Viewer focuses on playback, not interactive measurement or analysis tools.

## Next steps
- Add a scenario/config loader to define bodies and models without code changes.
- Expand model library (aero drag, attitude control, contact constraints).
- Add automated regression tests and a reproducible benchmark suite.
