# TR: previous architectural decisions

**Audience:** coding agents and developers working on `TR`.
**Describes:** the MAPL3 line, branch `release/MAPL-v3`.
**Verified against:** `TR` @ `f4da9bc` (2026-11-06).

### Tracer list and behaviour come from resource files
- Pattern: the tracers are the rows of `TR_table::` in `TR_Registry.rc`; each has its own `TR_GridComp---<name>.rc`, read in SetServices and again in `TR_init_tracer_spec_` (`TR_GridCompMod.F90`; `TR_Registry.rc`).
- Likely reason: add a tracer without code change.   *Inferred*

### Source phase and sink phase in a configurable order
- Pattern: `first_phase:` fixes `spec%run_order`; `TR_run_tracer_` loops over two phases (`TR_GridCompMod.F90`, `TR_init_tracer_spec_`, `TR_run_tracer_`).
- Likely reason: some tracers (e.g. age of air) need the source added before the sink, others after.   *Inferred*

### Removal code copied from the GMI and GOCART components
- Pattern: the GMI removal modules live in this repository (`DryDepositionGmiMod.F90`, `WetRemovalGmiMod.F90`, `TR_GravSetMod.F90`: headers say they were originally in GMI); GOCART schemes come from `Chem_Shared` (`TR_GridCompMod.F90`, `USE` lines).
- Likely reason: avoid a dependency on the GMI component.   *Inferred*
- Question for maintainers: why are both removal families selectable per tracer?

### Tracers are friendly fields, convection handled by `TR` on request
- Pattern: each tracer is friendly to `DYNAMICS`, `TURBULENCE` and its own component name, and to `MOIST` unless `GOCART_convection:` is true (`TR_GridCompMod.F90`, SetServices).
- Likely reason: when `TR` applies its own convective transport, the moist component must not also transport it.   *Inferred*

### Decay product handed to another tracer
- Pattern: a decaying tracer stores `tracer_decay`; a tracer with `src_mode: decay_of_another_species` uses it as source (`TR_GridCompMod.F90`, `TR_run_tracer_`).
- Likely reason: radioactive chains such as Rn222 to Pb210.   *Inferred*

### Run only when the run alarm rings
- Pattern: Run_ returns unless `RunAlarm` rings, then clears it (`TR_GridCompMod.F90`, Run_).
- Likely reason: allow a component step longer than the heartbeat (`TR_DT:` is read for the overpass masks).   *Inferred*
