# TR (passive tracers) architecture

**Audience:** coding agents and developers working on `TR`.
**Describes:** the MAPL3 line, branch `release/MAPL-v3`.
**Verified against:** `TR` @ `f4da9bc` (2026-11-06).
**Placement:** as in the GEOSgcm fixture (`components.yaml` @ `f50622d`), key `TR`:
`src/Components/@GEOSgcm_GridComp/GEOSagcm_GridComp/GEOSphysics_GridComp/@GEOSchem_GridComp/@TR`.
Other fixtures (for example GEOSadas, GEOSldas) may place this repository
elsewhere; paths inside this repository do not change.

Parent: the Chemistry gridded component. Its architecture document is `../../docs/ARCHITECTURE.md` in the fixture; it covers nested repositories and how children are added and run.
This document does not repeat the parent's content. The repository holds one gridded component module plus three helper modules, and no children.

> **Build status on this line:** not built, not ported. On `release/MAPL-v3`, `../CMakeLists.txt`
> (`@GEOSchem_GridComp/CMakeLists.txt`) builds only `GOCART` (`esma_add_subdirectories(GOCART)`, under the comment
> "Restore subdirs as components are ported to MAPL3."); `TR` appears only in commented-out lines (the subdirectory list and the `DEPENDENCIES` of the commented-out library block). The code in this
> repository is the MAPL2 code, kept for a later port. `snk_horiz: masks` aborts on this line ("TRACER snk MASKS not implemented yet", `TR_init_tracer_spec_`). The Chemistry top-level library (`GEOS_ChemGridComp.F90`, which adds
> this component) is present but not compiled on this line: the `esma_add_library` block that builds it is commented out in
> `../CMakeLists.txt`. Statements about it describe the file as it is. `Chem_Shared`, which `TR` lists as a dependency, is
> not built on this line either (the `Shared` directory is in the commented-out list).

## Directory and file map

| Path | Content |
|---|---|
| `TR_GridCompMod.F90` | Module `TR_GridCompMod`, the only public name is `SetServices`; all component logic, derived types, per-tracer source/sink code |
| `DryDepositionGmiMod.F90` | Module `DryDepositionGmiMod`, public `TR_GMI_DryDeposition`; dry deposition copied from the GMI chemistry component |
| `WetRemovalGmiMod.F90` | Module `WetRemovalGmiMod`, public `TR_GMI_WetRemoval`; in-cloud scavenging and below-cloud washout, copied from GMI |
| `TR_GravSetMod.F90` | Module `TR_GravSetMod`, public `TR_GMI_GravitationalSettling`; gravitational settling of aerosols |
| `TR_GridComp.rc` | Component-level resource file; one key, `strict_tracer_timing:` |
| `TR_Registry.rc` | List of tracers to run: table `TR_table::` (name, units, long name) |
| `TR_GridComp---<tracer>.rc` | One resource file per tracer (41 files on this line); source, sink and removal parameters |
| `TR_ExtData.yaml`, `TR_ExtData.rc` | ExtData (MAPL external data component) specifications of the files that feed the `SRC_2D_*`, `SRC_3D_*`, region-mask and vegetation imports; yaml and legacy rc forms |
| `CMakeLists.txt` | Builds library `TR` from the four `.F90` files; installs every `*.rc`, `*.yaml` into `etc` |
| `ChangeLog`, `CHANGELOG.md`, `README.md`, `.circleci`, `.github` | Housekeeping; `README.md` is one line |

The `.F90` module sizes are about 5,400 lines (`TR_GridCompMod.F90`) and about 1,000 lines each for the two GMI-derived modules.

## 1. Purpose

`TR` implements passive tracers: three-dimensional mixing-ratio fields that the model
advects and mixes, to which this component adds configurable sources and sinks (age of air,
radioactive decay chains, regional carbon monoxide, and similar), and optionally dry and wet removal
(`TR_GridCompMod.F90`, module header; `TR_Registry.rc`; `TR_GridComp---*.rc`).
The tracer list and each tracer's behaviour come from resource files, not from code.

## 2. Place in the hierarchy

- Parent is Chemistry (`GEOS_ChemGridComp.F90`). It adds `TR` as a child named `'TR'` when `ENABLE_TR` is true,
  with `MAPL_AddChild(GC, NAME='TR', SS=TR_SetServices, ...)` (`../GEOS_ChemGridComp.F90`, SetServices).
- `TR` has no children (`TR_GridCompMod.F90`, SetServices: no `MAPL_AddChild`).
- Parent connectivity into `TR` (`../GEOS_ChemGridComp.F90`, SetServices, `MAPL_AddConnectivity` with `DST_ID = TR`):
  from `CHEMENV`: `AIRDENS`, `AIRDENS_DRYP`, `DELP`, `CN_PRCP`, `NCN_PRCP`;
  from `GMICHEM`, only when both `TR` and `GMICHEM` are enabled: `OX` -> `OX_TR`, `stOX_loss` -> `stOX_loss`, `DD_OX` -> `DD_OX`.
- *Inferred:* the other imports (`T`, `PLE`, `ZLE`, `Q`, surface, convective-flux and boundary-layer fields)
  are supplied by connectivities made above Chemistry (Physics or the Atmosphere level) or by ExtData;
  this repository does not make them.
- The parent re-exports nothing from `TR` by explicit name that this review found (grep of `../GEOS_ChemGridComp.F90`
  for `TR` exports found only the connectivities above and the `TR` provider name in `GetProvider_`).
  The tracers leave `TR` as friendly internal fields (see State below).

## 3. Entry points

All set in `SetServices` with `MAPL_GridCompSetEntryPoint`: one Initialize, one Run phase, one Finalize
(`TR_GridCompMod.F90`, SetServices).

- **SetServices**: allocates the private `TR_State` and stores it with `ESMF_UserCompSetInternalState` under `'TR_STATE'`;
  reads `TR_Registry.rc` (table `TR_table::`) through `Runtime_RegistryCreate`; reads `strict_tracer_timing:` from `TR_GridComp.rc`;
  declares the fixed imports; then loops over the registry twice, loading each `TR_GridComp---<tracer>.rc`,
  first to add per-tracer imports and exports, then to add the tracer as an internal-state spec; ends with `MAPL_GenericSetServices`.
- **Initialize_**: `MAPL_GenericInitialize`; fills `TR_TracerKit` (index bounds, global sizes, latitudes and longitudes in degrees, longitude 0-360);
  calls `TR_init_tracer_spec_` once per tracer (reads all keys of the tracer's `.rc` into a `TR_TracerSpec`);
  builds the simple bundle `TR_PASSIVE_TRACERS` over the internal-state fields (`MAPL_SimpleBundleCreate`);
  allocates land-type arrays if any tracer uses `GMI_dry_deposition`; builds the 10 am and 2 pm local-overpass masks with `OVP_init`, `OVP_mask`
  (`TR_GridCompMod.F90`, Initialize_).
- **Run_**: returns at once unless the component's `RunAlarm` is ringing. Otherwise: decodes vegetation fraction (once) and leaf area index
  (each call) into the kit if any tracer uses `GMI_dry_deposition`; loops over tracers calling `TR_run_tracer_` (with a barrier around each when
  `strict_tracer_timing:` is true); fills export `TR_DELZ` from `ZLE`.
- **TR_run_tracer_** (internal routine, per tracer): loops over two phases in the order `spec%run_order` chosen by `first_phase:`
  (`source` or `sink`, set in `TR_init_tracer_spec_`).
  The source phase adds or replaces tracer mass by `src_mode:`
  (`constant`, `file2d`, `file3d`, `model_field`, `maintain_mixing_ratio`, `periodic`, `decay_of_another_species`, `none`),
  limited by `src_horiz:` / `src_vert:` and optional surface constraints.
  The sink phase applies `snk_mode:` (`constant`, `efolding`, `halflife`, `chemical_loss`, `none`), then, per the flags, GMI or GOCART settling, dry
  deposition, wet removal and convection. After both phases it fills the overpass exports `OVP10_<tracer>`, `OVP14_<tracer>`
  (`TR_GridCompMod.F90`, TR_run_tracer_, TR_init_tracer_spec_).
- **Finalize_**: deallocates per-tracer and kit arrays and the bundle, then `MAPL_GenericFinalize`.

Contained helper routines of `TR_run_tracer_` (`TR_GridCompMod.F90`): `impose_surface_constraints`, `TR_GMI_settle_and_depos`,
`TR_GMI_wet_removal`, `TR_GOCART_wet_removal`, `TR_GOCART_convection`, `TR_GOCART_settling`, `TR_GOCART_drydep`,
`set_regional_mask`, `set_bool_mask`.

## 4. State

- **Internal state**: one three-dimensional tracer per registry entry, added with `MAPL_AddInternalSpec` (`ADD2EXPORT=.TRUE.`, so each is
  also an export), `FRIENDLYTO = 'DYNAMICS:TURBULENCE[:MOIST]:<component name>'`; the `MOIST` entry is left out when the tracer's
  `GOCART_convection:` is true, because `TR` then applies convection itself (`TR_GridCompMod.F90`, SetServices). Chemistry gathers these into
  `CHEM_TRACERS` (see the parent's friendly-tracer handling in `../GEOS_ChemGridComp.F90`). Internal state is checkpointed by default; no `RESTART` option is set on the
  tracer specs.
- **Imports**: declared in SetServices. Fixed set: meteorology (`T`, `Q`, `PLE`, `ZLE`, `DELP`, `AIRDENS`, `AIRDENS_DRYP`, `TROPP`, `DQDT`, `RH2`),
  precipitation and convection (`PFL_CN`, `PFL_LSAN`, `PFI_LSAN`, `CN_PRCP`, `NCN_PRCP`, `CNV_MFD`, `CNV_MFC`, `QLCN`, `QICN`),
  surface and boundary layer (`LWI`, `T2M`, `Z0H`, `SWNDSRF`, `USTAR`, `FRLAKE`, `FROCEAN`, `FRACI`, `FRLANDICE`, `TSOIL1`, `SNOWDP`, `ZPBL`, `PPBL`, `SH`, `AREA`),
  `CLDTT`, and the GMI ozone link `OX_TR`, `DD_OX`. All are `RestartSkip` except `CLDTT`.
  Conditional imports: `SRC_2D_<tracer>` / `SRC_3D_<tracer>` (when `src_mode:` is `file2d` / `file3d`), one region-mask import per distinct `regions_ExtData_entry:`, `stOX_loss` (when a tracer has `snk_mode: chemical_loss` and `loss_species: OX`),
  and `TR_VEG_FRAC`, `TR_LAI_FRAC`.
- **Exports** besides the tracers: `TR_DELZ`; per tracer `OVP10_`, `OVP14_`, `STtend_`, `STtendVsum_`, `DDtend_`, `WRtend_`, `WRtendVsum_`, `CVtend_`,
  `CLtend_`, `CLtendVsum_`; and when the source or sink mode asks, `EM_`, `PRtend_` (for `decay_of_another_species`) and `DKtend_`
  (for `efolding`, `halflife`) (`TR_GridCompMod.F90`, SetServices).
  *Inferred:* no other repository in the Chemistry tree consumes them by name (the Chemistry grep found none); they are diagnostic exports for HISTORY.
- **Private Fortran state**: `TR_State` holds the registry, the `TR_TracerKit` shared across tracers (grid, land types) and an array of `TR_TracerSpec`
  (one per tracer); module-level `SAVE` variables hold the overpass masks (`TR_GridCompMod.F90`, type definitions).
- **Carry between tracers**: a tracer with a decay sink stores the lost amount in `spec%tracer_decay`; a tracer with
  `src_mode: decay_of_another_species` reads the precursor's `tracer_decay` (named by `src_species:`) scaled by molecular weight
  (`TR_GridCompMod.F90`, TR_run_tracer_). The precursor must therefore be earlier in the registry order, because the loop in Run_ is in registry order;
  *Inferred:* that dependency is not checked in code.

## 5. Configuration that changes structure

- `ENABLE_TR` in the parent's `GEOS_ChemGridComp.rc` (`../GEOS_ChemGridComp.rc`).
- `TR_Registry.rc`, table `TR_table::`: every row is run, and for each row `TR_GridComp---<name>.rc` must exist in the run directory
  (`TR_GridCompMod.F90`, SetServices; registry header comments). Shipped table: `aoa`, `e90`, `Rn222`, `CH3I`.
- Per-tracer file `TR_GridComp---<tracer>.rc` (examples `TR_GridComp---aoa.rc`, `TR_GridComp---Be10.rc`). Key families, all read in `TR_init_tracer_spec_`:
  - `first_phase:` (`source` | `sink`).
  - `src_mode:`, `src_value:`, `src_add:`, `src_horiz:`, `src_vert:` (and level or pressure bounds), `src_species:`, `src_field_name:`, `mw:`,
    `surface_constraint_count:` with `surf_con<N>_*` keys, `regions_ExtData_entry:`.
  - `snk_mode:`, `snk_value:` / `snk_period:`, `snk_horiz:`, `snk_vert:`, `snk_decays_to:`, `loss_species:`.
  - Removal schemes: `GMI_dry_deposition:`, `GMI_wet_removal:` and the `GMI_*` parameters; `GOCART_wet_removal:`, `GOCART_convection:`,
    `GOCART_settling:`, `GOCART_dry_deposition:` and the `GOCART_*` parameters. In `aoa` they are absent; in `Be10` the GOCART ones are on and the `GMI_*` lines are comments.
- `strict_tracer_timing:` in `TR_GridComp.rc` (diagnostic barriers only).
- The restart file names come from the parent's `TR_INTERNAL_RESTART_FILE`, `TR_IMPORT_RESTART_FILE` (`../GEOS_ChemGridComp.rc`).

## 6. Dependencies

- Library `TR` links `MAPL GMAO_mpeu Chem_Shared GEOS_Shared esmf` (`CMakeLists.txt`). `Chem_Shared` is not built on this line (it is in the commented-out `Shared` entry of `../CMakeLists.txt`).
- From `Chem_Shared`: `WetRemovalMod` (`WetRemovalGOCART`), `ConvectionMod` (`convection`, `set_vud`), `Chem_SettlingMod` (`Chem_Settling2`),
  `DryDepositionMod` (`DryDepositionGOCART`), `VegLaiMod` (`Decode_Land_Types`, `Decode_XLAI`), `GmiTimeControl_mod` (`GetSecondsFromJanuary1`, used in `TR_run_tracer_`), `SZA_from_MAPL_mod` (`compute_SZA`, used in `TR_GMI_settle_and_depos`) (`TR_GridCompMod.F90`, `USE` lines; the last two are in `../Shared/Chem_Shared`).
  Also `Chem_Mod`, `Chem_ArrayMod`, `Runtime_RegistryMod`, `Chem_UtilMod`, `OVP` (overpass masks), `m_inpak90`, `m_die`.
- The three `*Gmi*` and `TR_GravSetMod` modules are local to this repository; they do not use the GMI component's code.
- ExtData supplies the source files; this repository ships the specification (`TR_ExtData.yaml`).

## 7. Known traps

- The `TR_GMI_*` and `TR_GOCART_*` removal branches are selected per tracer by flags in its `.rc`, not by component-level options; both families
  can be set on one tracer.
- Removal exports (`STtend_`, `DDtend_`, `WRtend_`, `CVtend_`, `CLtend_`) are declared for every tracer regardless of the flags: the `IF` guards in
  SetServices are commented out. `lai_needed` is therefore always true, so `TR_VEG_FRAC` and `TR_LAI_FRAC` are always imported, though they are only decoded in Run_ when some tracer has `GMI_dry_deposition:`.
- Adding a tracer needs a registry row *and* a `TR_GridComp---<name>.rc`; for `file2d` / `file3d` sources also an ExtData entry
  `SRC_2D_<name>` / `SRC_3D_<name>`; for region constraints an ExtData mask entry (`regions_ExtData_entry:`).
- Units: the code distinguishes mass (`kg kg-1`) and volume (`mol mol-1`) mixing ratio from the registry `units` string only; any other string is
  `OTHER_UNITS` (`TR_init_tracer_spec_`).
- A tracer's friendly list drops `MOIST` when `GOCART_convection:` is true; changing that flag changes which parent applies convection to the tracer.
- Run_ does nothing when the run alarm is not ringing, so tracer sources and sinks apply only at the component's own time step.

## 8. Decisions

See [decisions.md](decisions.md).

## 9. Observations (noted, not fixed) and open questions

- `TR_Registry.rc` has a second table `pre_configured::` that no `.F90` file in this repository reads (grep for `pre_configured`: no hits);
  it is a longer list that starts with the same four rows (`aoa`, `e90`, `Rn222`, `CH3I`) and adds `Pb210`, `nh_5`, `nh_50`, `aoa_nh`, `st80_25`, `CO_25`, `CO_50`, and others (`TR_Registry.rc`). Question for maintainers: is it read by an external tool or script?
- Both `TR_ExtData.yaml` and `TR_ExtData.rc` are installed; the comments in the per-tracer files refer to `TR_ExtData.rc`.
  Question: which is the current source of truth in the GEOSgcm fixture?
- SetServices comments out the `IF` guards on the removal exports (see Known traps); the `GMI_dry_deposition` and `GMI_wet_removal` values are read but unused there.
- `OX_TR`, `DD_OX` are imported unconditionally, while `stOX_loss` is imported only when needed; the Chemistry connectivity to `OX_TR` is made only if `GMICHEM`
  is enabled. *Unverified:* what ExtData or default fills `OX_TR` when `GMICHEM` is off (the code comment says "Add these entries to ExtData.rc").
- The Run_ guard against negative mixing ratios (`where(q < 0) q = 1.0e-20`) sits under `#if (0)`.
- Where `TR_run_tracer_` needs a precursor tracer to run before a dependent one, order relies on registry order (see State above).
