# TR (passive tracers) - agent guide

**Describes:** the MAPL2 line, branch `develop`. **Verified against:** `TR` @ `9e895f9` (2026-11-06).

## What this is
`TR` is the GEOS passive-tracer gridded component, one of the children of the Chemistry
gridded component (`GEOS_ChemGridComp.F90`, added when `ENABLE_TR` is true). It has no children.
The tracers are chosen by resource files; the code is generic.

## Placement (GEOSgcm fixture, `components.yaml` @ `0f2e0fb`, key `TR`)
`src/Components/@GEOSgcm_GridComp/GEOSagcm_GridComp/GEOSphysics_GridComp/@GEOSchem_GridComp/@TR`.
Parent: Chemistry `AGENTS.md` (`../AGENTS.md` in the fixture), Chemistry architecture (`../docs/ARCHITECTURE.md` in the fixture).

## Documents
| File | Content |
|---|---|
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | File map, entry points, state, per-tracer resource format, traps, observations |
| [docs/decisions.md](docs/decisions.md) | Recorded design patterns, with likely reasons |

## Where things are
- `TR_GridCompMod.F90`: whole component (SetServices, Initialize_, Run_, Finalize_, `TR_init_tracer_spec_`, `TR_run_tracer_`).
- `DryDepositionGmiMod.F90`, `WetRemovalGmiMod.F90`, `TR_GravSetMod.F90`: removal code copied from GMI.
- `TR_Registry.rc`: which tracers run. `TR_GridComp---<tracer>.rc`: how each behaves. `TR_ExtData.yaml`: input files.

## Rules for agents
- Note bugs and oddities; do not fix them unless asked.
- Adding a tracer: registry row, per-tracer `.rc`, and ExtData entries for any `file2d`, `file3d` or `mask` input.
- Tracers are friendly to `DYNAMICS`, `TURBULENCE`, usually `MOIST`; Chemistry gathers them into `CHEM_TRACERS`. Do not change the friendly list without reading the Chemistry document.
- Keep these documents current when behaviour changes. Cite file and routine, no line numbers.

## Build and test
Built as library `TR` from `CMakeLists.txt` as part of GEOSgcm (needs MAPL, `Chem_Shared`, `GEOS_Shared`); installs all `*.rc`, `*.yaml` to `etc`.
No tests live in this repository; exercise it through a GEOSgcm run with `ENABLE_TR: .TRUE.`.
