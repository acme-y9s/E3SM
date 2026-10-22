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

The first state-layer changes add pool-compatible C, N, and P column-reservoir
arrays. They are not active in transport yet. Before activation, the work must
also add initialization, restart registration, C-13/C-14 companion state, and
overflow transfer logic.

## Planned deep-reservoir sequence

1. Initialize new reservoir pools to zero for startup and safely handle missing
   restart variables.
2. Add restart registration with interpolation support.
3. Add C-13 and C-14 companion pools.
4. Transfer deepest-layer excess into the reservoir while preserving pool
   composition and C/N/P/isotope conservation.
5. Add history diagnostics for reservoir mass and inflow.
6. Test disabled and enabled modes against the existing fixed-grid baseline.

The deep reservoir remains disabled until all state, restart, and transport
pieces are complete.

## Validation requirements

For the disabled mode, results should remain unchanged relative to the current
fixed-grid implementation. For the enabled mode, verify:

- restart continuation gives consistent reservoir values;
- C, N, P, C-13, and C-14 are conserved;
- the conservation residual remains near zero;
- the deepest fixed-layer overflow is represented in reservoir inflow;
- the new diagnostics appear only when requested by the history namelist.
