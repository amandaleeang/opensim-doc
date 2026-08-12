# Developer Guide

This guide helps you **read, navigate, and modify** the OpenSimulator source code.
It assumes you are comfortable with C# and want to know *where things live* so you can
fix a bug or build a feature without getting lost.

All paths below are relative to the `OpenSim/` directory of the source tree
(e.g. `OpenSim/Region/Application/...`).

## Quick mental model

OpenSim is **two programs** that talk over HTTP:

- **OpenSim** (the *simulator*) – hosts regions ("scenes") and talks to viewers.
- **Robust** (the *grid services server*) – hosts shared services (assets,
  inventory, grid map, login, presence, …) as HTTP endpoints.

Both are built from many small `.csproj` projects (one per folder), orchestrated by
`prebuild.xml` at the repo root – not one giant project.

## Sections

1. [Architecture overview](architecture.md) – entry points, the module system, and
   how services vs connectors work.
2. [Where to look](where-to-look.md) – a "I want to change X → start here" index.
3. [Data flow](data-flow.md) – walkthroughs of login and region teleport/crossing.

## Before you dig in

- Build it first: see the [Admin – build from source](../admins/build-from-source.md).
- The code uses **Mono.Addins** for plugin discovery; modules/plugins are wired
  through extension points rather than hardcoded `new` calls.
- `scene.RegisterModuleInterface<T>(this)` / `scene.RequestModuleInterface<T>()` is
  the dependency-injection pattern used everywhere – learn it early.
