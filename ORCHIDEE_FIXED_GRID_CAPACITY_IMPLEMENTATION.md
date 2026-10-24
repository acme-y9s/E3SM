# ORCHIDEE-Inspired Fixed-Grid Peatland Implementation

This file records the development of the optional ORCHIDEE-inspired features on
branch `y9s/orchidee-fixed-grid-capacity`.

## Baseline

The implementation retains ELM's fixed soil layers. It does not move the ELM
hydrology or thermal layer boundaries.

The colleague's peat-density implementation remains controlled by:

```fortran
use_peatland_compaction_profile
```

Because ELM's namelist logic derives this from
`use_peatland_vertical_transport` when not explicitly specified, cases using
`use_humhol = .true.` may receive the compaction profile automatically.

## Completed features

The branch includes the following diagnostic and conservation features:

- Fixed-layer peat carbon capacity.
- Geometric peat fraction for each fixed layer.
- Deepest fixed layer containing solid peat carbon.
- Layer-level carbon overflow diagnostics.
- Carbon-conservation residual diagnostics.
- Runtime flag `use_peatland_fixed_grid_capacity`.
- OpenACC, MPI, namelist, and validation support for that flag.

The five main diagnostics are:

```text
PEAT_C_CAPACITY
PEAT_LAYER_PEAT_FRACTION
PEAT_C_OVERFLOW
PEAT_DEEPEST_C_DEPTH
PEAT_C_CONSERVATION_RESIDUAL
```

These are diagnostics, not restart state. They are recalculated from the
current model state.

## Current deep-reservoir work

The optional control flag is:

```fortran
use_peatland_deep_effective_reservoir = .false.
```

It requires:

```fortran
use_peatland_fixed_grid_capacity = .true.
```

The implementation now includes pool-compatible C, N, and P column-reservoir
arrays, with C-13 and C-14 companion state when isotope options are enabled.
The reservoir state is initialized safely, registered in restart files, and
uses pool-resolved names:

```text
peat_deep_reservoir_c12_<pool>
peat_deep_reservoir_n_<pool>
peat_deep_reservoir_p_<pool>
```

The deepest fixed-layer transport boundary is open only when the deep-reservoir
flag is true. The net loss from each transported pool is transferred into the
corresponding reservoir. When the flag is false, ELM's original closed lower
boundary is retained.

The transfer is implemented for C, N, P, C-13, and C-14. Reservoir updates are
pool-resolved and restart-compatible.

## Planned deep-reservoir sequence

1. Initialize new reservoir pools to zero for startup and safely handle missing
   restart variables. **Completed.**
2. Add restart registration with pool-resolved names. **Completed.**
3. Add C-13 and C-14 companion pools. **Completed.**
4. Transfer deepest-layer material into the reservoir while preserving pool
   composition and C/N/P/isotope pathways. **Completed.**
5. Add history diagnostics for reservoir mass and boundary inflow. **Completed.**
6. Test disabled and enabled modes against the existing fixed-grid baseline.
   **In progress.**

The deep reservoir remains opt-in through the runtime flag.

## Deep-reservoir history diagnostics

Deepest-boundary flux diagnostics are column-level, with units of mass per
area per second:

```text
PEAT_DEEP_BOUNDARY_C_FLUX
PEAT_DEEP_BOUNDARY_N_FLUX
PEAT_DEEP_BOUNDARY_P_FLUX
PEAT_DEEP_BOUNDARY_C13_FLUX
PEAT_DEEP_BOUNDARY_C14_FLUX
```

Cumulative pool-summed reservoir diagnostics are also available as column-level
history fields:

```text
PEAT_DEEP_RESERVOIR_C
PEAT_DEEP_RESERVOIR_N
PEAT_DEEP_RESERVOIR_P
```

These diagnostics are registered by ELM but must be requested in the desired
`hist_fincl` list, normally `hist_fincl2` because they have a `column`
dimension. They are not restart variables; the pool-resolved reservoir fields
in `ColumnDataType.F90` are the restart state.

## Validation requirements

For the disabled mode, results should remain unchanged relative to the current
fixed-grid implementation. For the enabled mode, verify:

- restart continuation gives consistent reservoir values;
- C, N, P, C-13, and C-14 are conserved;
- the conservation residual remains near zero;
- the deepest fixed-layer overflow is represented in reservoir inflow;
- the new diagnostics appear only when requested by the history namelist.

## Pathfinder validation completed

On Pathfinder, a fresh enabled case was run with:

```fortran
use_peatland_fixed_grid_capacity = .true.
use_peatland_deep_effective_reservoir = .true.
```

The model successfully read the year-19 restart and completed a continuation
through year 37. The restart files contained pool-resolved C, N, and P
reservoir variables, and their values increased after continuation. The h1
history files contained all five deepest-boundary flux diagnostics and the
three cumulative reservoir diagnostics. C-13 and C-14 fluxes were zero in the
test because the test case did not produce nonzero isotope transport.
