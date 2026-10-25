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
   **Completed for the deep-reservoir pathway.**

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

The conservation-residual diagnostics were also enabled in the h1 output:

```text
PEAT_DEEP_RESERVOIR_C_RESIDUAL
PEAT_DEEP_RESERVOIR_N_RESIDUAL
PEAT_DEEP_RESERVOIR_P_RESIDUAL
```

They remained finite and effectively zero at all checkpoints (years 1, 10,
19, 28, and 37). The largest observed magnitudes were approximately
`2.1e-26 gC/m2` for C, `1.8e-28 gN/m2` for N, and `3.4e-29 gP/m2` for P.
This validates the deepest-boundary reservoir balance through both the initial
18-year run and the 18-year restart continuation.

## Deep-reservoir turnover

The optional first-order turnover pathway releases a fraction of each
pool-resolved deep-reservoir component back into the deepest active ELM soil
layer. It is controlled by the namelist switch:

```fortran
use_peatland_deep_reservoir_turnover = .true.
```

The turnover coefficient is a scientific parameter stored in the ELM parameter
NetCDF file and modified through the OLMT `[parameters]` section. For the
Pathfinder test, the configuration used:

```ini
peat_deep_reservoir_turnover_rate = 0.001
```

The release fraction is calculated with a first-order update,
`1 - exp(-rate * dt_years)`, and is applied separately to C, N, and P pools.
The reservoir subtraction and deepest-layer addition use the same amount, so
the reservoir-plus-column total is conserved. The default rate is zero, and
the turnover switch is opt-in.

Column-level time-mean turnover flux diagnostics are available when requested
in the history namelist:

```text
PEAT_DEEP_RESERVOIR_TURNOVER_C
PEAT_DEEP_RESERVOIR_TURNOVER_N
PEAT_DEEP_RESERVOIR_TURNOVER_P
```

They have units of `gC/m2/s`, `gN/m2/s`, and `gP/m2/s`, respectively. Positive
values indicate release from the deep reservoir into the deepest active layer.

### Turnover validation on Pathfinder

Case:

```text
20261024_US-SPR_ICB1850CNRDCTCBC_ad_spinup_turnover
```

The case completed an initial 18-year run followed by an 18-year restart
continuation, with both reservoir and turnover switches enabled. The restart
was read at year 19 and the continuation completed through year 37.

The turnover diagnostics increased between years 19 and 37, confirming active
release. At year 37, representative active-column values were approximately:

```text
C: 1.6e-22 to 2.1e-19 gC/m2/s
N: 7.6e-24 to 5.7e-21 gN/m2/s
P: 1.2e-23 to 1.9e-22 gP/m2/s
```

The reservoir conservation residuals at year 37 remained near machine
precision, approximately `1e-26` for C, `1e-28` for N, and `1e-30` for P.
This validates turnover activation, restart compatibility, and C/N/P
conservation across the 36-year test sequence.
