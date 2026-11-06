# TR (passive tracers) - agent guide

**Describes:** the MAPL3 line, branch `release/MAPL-v3`. **Verified against:** `TR` @ `f4da9bc` (2026-11-06).

> **Build status on this line:** not built, not ported. On `release/MAPL-v3`, `../CMakeLists.txt`
> (`@GEOSchem_GridComp/CMakeLists.txt`) builds only `GOCART` (`esma_add_subdirectories(GOCART)`, under the comment
> "Restore subdirs as components are ported to MAPL3."); `TR` appears only in commented-out lines (the subdirectory list and the `DEPENDENCIES` of the commented-out library block). The code in this
> repository is the MAPL2 code, kept for a later port. `snk_horiz: masks` aborts on this line ("TRACER snk MASKS not implemented yet", `TR_init_tracer_spec_`). The Chemistry top-level library (`GEOS_ChemGridComp.F90`, which adds
> this component) is present but not compiled on this line: the `esma_add_library` block that builds it is commented out in
> `../CMakeLists.txt`. Statements about it describe the file as it is. `Chem_Shared`, which `TR` lists as a dependency, is
> not built on this line either (the `Shared` directory is in the commented-out list).

## What this is
`TR` is the GEOS passive-tracer gridded component, one of the children of the Chemistry
gridded component (`GEOS_ChemGridComp.F90`, added when `ENABLE_TR` is true). It has no children.
The tracers are chosen by resource files; the code is generic.

## Placement (GEOSgcm fixture, `components.yaml` @ `f50622d`, key `TR`)
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
- Adding a tracer: registry row, per-tracer `.rc`, and ExtData entries for any `file2d` or `file3d` input (and for region masks named by `regions_ExtData_entry:`).
- Tracers are friendly to `DYNAMICS`, `TURBULENCE`, usually `MOIST`; Chemistry gathers them into `CHEM_TRACERS`. Do not change the friendly list without reading the Chemistry document.
- Keep these documents current when behaviour changes. Cite file and routine, no line numbers.

## Build and test
Not built on this line (see build status above). `CMakeLists.txt` defines library `TR` (lists `MAPL`, `GMAO_mpeu`, `Chem_Shared`, `GEOS_Shared`, `esmf`) and installs all `*.rc`, `*.yaml` to `etc`.
No tests live in this repository; on the MAPL2 line, exercise it through a GEOSgcm run with `ENABLE_TR: .TRUE.`; on this line it cannot be run until ported.
