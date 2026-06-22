# WayU

**Camera-based indoor positioning and accessibility-first wayfinding for large campuses — localize from a single photo, then route step-free to any door.**

![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.115-009688?logo=fastapi&logoColor=white)
![PyTorch](https://img.shields.io/badge/PyTorch-2.3-EE4C2C?logo=pytorch&logoColor=white)
![OpenCV](https://img.shields.io/badge/OpenCV-4.10-5C3EE8?logo=opencv&logoColor=white)
![COLMAP](https://img.shields.io/badge/COLMAP%20%2F%20hloc-SfM-555555)
![PostGIS](https://img.shields.io/badge/PostGIS%20%2B%20pgRouting-336791?logo=postgresql&logoColor=white)
![React](https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=black)
![TypeScript](https://img.shields.io/badge/TypeScript-5.6-3178C6?logo=typescript&logoColor=white)
![Vite](https://img.shields.io/badge/Vite-5-646CFF?logo=vite&logoColor=white)
![Swift](https://img.shields.io/badge/Swift-6-F05138?logo=swift&logoColor=white)
![ARKit](https://img.shields.io/badge/ARKit%20%2F%20RealityKit-iOS%2017-000000?logo=apple&logoColor=white)

---

## What it does

GPS dies indoors and degrades between dense buildings, which is exactly where people get lost — and where wheelchair users, low-vision travelers, and first-time visitors most need reliable directions. WayU is a **Visual Positioning System (VPS)** for campuses: it figures out where you are and which way you are facing from a single camera frame, with no beacons, no Wi-Fi fingerprinting, and no floor stickers — then guides you to your destination along a route that respects how *you* move through the world.

The whole experience is built around one loop:

> **localize → route → guide**

Point your phone, get a centimeter-to-meter-scale pose, ask for a destination, and follow an accessibility-aware path rendered straight into the live camera view. "Take me to the lab without stairs" returns a provably step-free route — or an honest "that destination isn't mapped yet" instead of a confident hallucination.

## How it works

WayU is a monorepo with three cooperating apps over one spatial backend.

```
  iOS app (ARKit)            React dashboard
  capture · localize         build maps · localize
  AR turn-by-turn            inspect routes · analytics
        \                         /
         \                       /
          v                     v
        ┌───────────────────────────┐
        │   FastAPI spatial backend  │
        │                            │
        │  CV   localize / map / SfM │  classical (CPU) ↔ hloc/COLMAP (GPU)
        │  GEO  IMDF indoor maps     │  OGC IMDF subset, GeoJSON exchange
        │  NAV  accessibility router │  A* (in-proc) ↔ pgRouting (DB)
        │  AI   grounded assistant   │  natural-language → grounded route
        └───────────────────────────┘
              │                 │
        SQLite + networkx   PostGIS + pgRouting
        (zero-setup demo)   Redis · object store (production)
```

**1. Mapping.** A zone is reconstructed into a metric 3D model — sparse points, reference keyframes, and per-point descriptors. The same `LocalMap` structure is produced two ways behind one interface: a synthetic reconstructor that lets the entire pipeline run and be benchmarked on CPU before any footage exists, and a real **Structure-from-Motion** backend (COLMAP / hloc with learned features) for production maps on a GPU rig.

**2. Localization.** A query frame goes through retrieval → descriptor matching → **PnP + RANSAC** to recover a full 6-DoF camera pose. The localizer is descriptor-agnostic by design: it runs on classical features (SIFT/ORB) on CPU today and swaps to learned features (e.g. SuperPoint) unchanged when the GPU backend is enabled.

**3. Routing.** Indoor geometry is stored as a pragmatic **IMDF** (Indoor Mapping Data Format) subset and rendered as standard GeoJSON, while a separate routable graph carries accessibility attributes in the **OpenStreetMap vocabulary** (`incline`, `surface`, `width`, `kerb`, `tactile_paving`, …) so it interoperates with OSM tooling. The same cost model runs over networkx A* in-process or pgRouting in PostGIS, identically.

**4. Assistant.** A grounded natural-language layer parses requests like "guide me to the library, no stairs" into routing calls. The model phrases the answer but never invents buildings, rooms, or accessibility facts — every spatial claim is sourced from the campus database and the route engine, with a deterministic fallback when no model is configured.

## Highlights

- **Single-image 6-DoF localization** — no GPS, beacons, or Wi-Fi fingerprinting; pose recovered geometrically from one camera frame.
- **Accessibility as a first-class constraint, not a filter.** Routes encode ADA gating values as *hard exclusions* (running slope ≤ 1:20, ramp slope ≤ 1:12, cross slope ≤ 1:48, min clear width 815 mm) so an "accessible" route is provably conformant — plus *soft penalties* that steer away from heavy doors, rough surfaces, and mild grades. Profiles cover fastest-walk, step-free/wheelchair, and low-vision (tactile-preferring) travel.
- **Privacy-by-design ingest.** People, faces, and license plates are masked *before* any reconstruction or feature extraction, so permanent maps are never built from people pixels — treated as a correctness and trust requirement, not an optimization.
- **Pluggable CV backends behind one interface.** The demo runs end-to-end on CPU with zero external services; flipping two env vars promotes the exact same pipeline to the GPU SfM stack.
- **Live AR guidance on iOS** — ARKit + RealityKit render turn-by-turn directions and a minimap into the camera feed, with on-device capture and frame encoding.
- **Operator dashboard** — a React/TypeScript control surface to build maps, run localization, inspect routes, and view analytics and reports.
- **Honest evaluation built in.** An Aachen-style benchmark generates held-out query poses with known ground truth and reports thresholded success rates plus median/percentile translation and rotation error and latency — the accuracy gate runs as a first-class CLI command, not an afterthought.

## Architecture notes

- **One backend, two deployment tiers.** The dev/demo tier is SQLite + networkx + in-process FAISS with no containers; the production tier brings up PostGIS + pgRouting, Redis, and an S3-compatible object store via Docker Compose — same code paths, swapped implementations.
- **Standards-first data model.** IMDF (OGC community standard, GeoJSON / RFC 7946) for indoor maps and the OSM accessibility vocabulary for the routing graph keep the data portable rather than locked to a bespoke schema.
- **Heavy reconstruction runs on a GPU rig / SLURM HPC cluster** for production map builds; everything else is designed to run on a laptop.
- **Token-authenticated, rate-limited localization API** with auto-generated OpenAPI docs.

## Tech stack

| Layer | Technologies |
|-------|--------------|
| **Backend** | Python 3.12, FastAPI, Pydantic v2, SQLAlchemy 2.0, Typer (CLI) |
| **Computer vision** | NumPy, OpenCV, PyTorch, COLMAP / hloc (SfM + learned features), FAISS |
| **Routing & geo** | networkx (A*), Shapely, IMDF, PostGIS + pgRouting |
| **Assistant** | Grounded LLM layer with deterministic fallback |
| **Dashboard** | React 18, TypeScript, Vite |
| **iOS** | Swift 6, SwiftUI, ARKit, RealityKit, CoreMotion |
| **Infrastructure** | Docker Compose, PostgreSQL/PostGIS, Redis, S3-compatible object store |

## Evaluation

WayU treats accuracy as something you measure, not assert. Because the synthetic reconstruction carries true ground-truth poses, the localization engine is benchmarked directly: held-out queries are sampled across the mapped volume and scored with the standard `visuallocalization.net` thresholds (translation/rotation tolerances), reporting success rate, median and 90th-percentile error, and latency. Real-footage accuracy is validated separately against surveyed checkpoints on the production rig. Specific figures depend on the zone and capture and are reported per-build rather than baked into this overview.

## Status

Active development. The end-to-end loop — map build, single-frame localization, accessibility routing, and the grounded assistant — runs today on CPU with zero external services, and the production CV/data/routing path (GPU SfM, PostGIS + pgRouting) is wired behind the same interfaces. The iOS AR client and operator dashboard exercise the live `localize → route → guide` loop against the backend.

> Source code is maintained in a private repository; this page is a public overview of the system.
