# Data model

This document describes the `"engineering"` data model type in PowerModelsDistribution, which is transformed at runtime, or at the user's direction into a `"mathematical"` data model for optimization.

In this document,

- `nphases` refers to the number of non-neutral, non-ground active phases connected to a component,
- `nconductors` refers to all active conductors connected to a component, _i.e._ `length(connections)`, and
- `nwindings` refers to the number of windings of a transformer.

The data structure is in the following format

```julia
Dict{String,Any}(
    "component_type" => Dict{Any,Dict{String,Any}}(
        id => Dict{String,Any}(
            "parameter" => value,
            ...
        ),
        ...
    ),
    ...
)
```

Valid component types are those that are documented in the sectios below. Each component object is identified by an `id`, which can be any immutable value (`id <: Any`), but `id` does not appear inside of the component dictionary, and only appears as keys to the component dictionaries under each component type.

Each edge or node component (_i.e._ all those that are not data objects or buses), are expected to have `status` fields to specify whether the component is active or disabled, `bus` or `f_bus` and `t_bus`, to specify the buses that are connected to the component, and `connections` or `f_connections` and `t_connections`, to specify the terminals of the buses that are actively connected in an ordered list. Terminals/connections can be any immutable value, as can bus ids.

Parameter values on components are expected to be specified in SI units by default (where applicable) in the engineering data model. Relevant expected units are noted in the sections below. Where units is listed as watt, real units will be watt * `v_var_scalar`. Where units are listed as volt/var, real units will be volt/var * `v_var_scalar`, and multiplied by `vnom`, where that value exists.

The Used column describes the situtations where certain parameters are used. "always" indicates those values are used in all contexts, `opf`, `mld`, or any other problem name abbreviation indicate they are used in particular for those problems. "solution" indicates that those parameters are outputs from the solvers. "multinetwork" indictes these values are only used to build multinetwork problems.

Those parameters that have a default may be omitted by the user from the data model, they will be populated by the specified default values.

Components that support "codes", such as lines, switches, and transformers, behave such that any property on said object that conflicts with a value in the code will override the value given in the code object. This is noted on each object where this is relevant.

## Root-Level Properties

At the root level of the data structure, the following fields can be found.

| Name         | Default         | Type                 | Used   | Description                                                                                                                |
| ------------ | --------------- | -------------------- | ------ | -------------------------------------------------------------------------------------------------------------------------- |
| `name`       |                 | `String`             |        | Case name                                                                                                                  |
| `data_model` | `"engineering"` | `String`             | always | `"engineering"`, `"mathematical"`, or `"dss"`. Type of the data model (this document describes `data_model="engineering"`) |
| `settings`   | `Dict()`        | `Dict{String,<:Any}` | always | Base settings for the data model, see Settings section below for details                                                   |

## Settings (`settings`)

At the root-level of the data model a `settings` dictionary object is expected, containing the following fields.

| Name             | Default | Type   | Units | Used   | Description                                                    |
| ---------------- | ------- | ------ | ----- | ------ | -------------------------------------------------------------- |
| `v_var_scalar`   | `1e3`   | `Real` |       | always | Scalar for voltage values                                      |
| `vbase`          |         | `Real` |       | always | Voltage base (_i.e._ basekv) at `base_bus`                     |
| `sbase`          |         | `Real` |       | always | Power base (baseMVA) at `base_bus`                             |
| `base_bus`       |         | `Any`  |       | always | id of bus at which `vbase` and `sbase` apply                   |
| `base_frequency` | `60.0`  | `Real` | Hz    | always | Frequency base, _i.e._ the base frequency of the whole circuit |

## Buses (`bus`)

The data model below allows us to include buses of arbitrary many terminals (_i.e._, more than the usual four). This would be useful for

- underground lines with multiple neutrals which are not joined at every bus;
- distribution lines that carry several conventional lines in parallel (see for example the quad circuits in NEVTestCase).

| Name             | Default     | Type            | Units  | Used         | Description                                                                                                                          |
| ---------------- | ----------- | --------------- | ------ | ------------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| `terminals`      | `[1,2,3,4]` | `Vector{Any}`   |        | always       | Terminals for which the bus has active connections                                                                                   |
| `vm_lb`          |             | `Vector{Real}`  | volt   | opf          | Minimum conductor-to-ground voltage magnitude, `size=nphases`                                                                        |
| `vm_ub`          |             | `Vector{Real}`  | volt   | opf          | Maximum conductor-to-ground voltage magnitude, `size=nphases`                                                                        |
| `vm_pair_ub`     |             | `Vector{Tuple}` |        | opf          | _e.g._  `[(1,2,210)]` means \|U1-U2\|>210                                                                                            |
| `vm_pair_lb`     |             | `Vector{Tuple}` |        | opf          | _e.g._  `[(1,2,230)]` means \|U1-U2\|<230                                                                                            |
| `grounded`       | `[]`        | `Vector{Any}`   |        | always       | List of terminals which are grounded                                                                                                 |
| `rg`             | `[]`        | `Vector{Real}`  |        | always       | Resistance of each defined grounding, `size=length(grounded)`                                                                        |
| `xg`             | `[]`        | `Vector{Real}`  |        | always       | Reactance of each defined grounding, `size=length(grounded)`                                                                         |
| `vm`             |             | `Vector{Real}`  | volt   | always       | Voltage magnitude at bus. If set, voltage magnitude at bus is fixed                                                                  |
| `va`             |             | `Vector{Real}`  | degree | always       | Voltage angle at bus. If set, voltage angle at bus is fixed                                                                          |
| `status`         | `1`         | `Int`           |        | always       | `1` or `0`. Indicates if component is enabled or disabled, respectively                                                              |
| `{}_time_series` |             | `Any`           |        | multinetwork | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for `status`, `vm`, `va`, `vm_lb`, `vm_ub` |

Each terminal `c` of the bus has an associated complex voltage phasor `v[c]`. There are two types of voltage magnitude bounds. The first type bounds the voltage magnitude of each `v[c]` individually,

- `lb <= |v[c]| <= ub`

However, especially in four-wire networks, bounds are more naturally imposed on the difference of two terminal voltages instead, e.g. for terminals `c` and `d`,

- `lb <= |v[c]-v[d]| <= ub`

This is why we introduce the fields `vm_pair_lb` and `vm_pair_ub`, which define bounds for pairs of terminals,

- $\forall$ `(c,d,lb)` $\in$ `vm_pair_lb`: `|v[c]-v[d]| >= lb`
- $\forall$ `(c,d,ub)` $\in$ `vm_pair_ub`: `|v[c]-v[d]| <= ub`

Finally, we give an example of how grounding impedances should be entered. If terminal `4` is grounded through an impedance `Z=1+j2`, we write

- `grounded=[4]`, `rg=[1]`, `xg=[2]`

### Special Case: three-phase bus

For three-phase buses, instead of specifying bounds explicitly for each pair of windings, often we want to specify 'phase-to-phase', 'phase-to-neutral' and 'neutral-to-ground' bounds. This can be done conveniently with a number of additional fields. First, `phases` is a list of the phase terminals, and `neutral` designates a single terminal to be the neutral.

- The bounds `vm_pn_lb` and `vm_pn_ub` specify the same lower and upper bound for the magnitude of the difference of each phase terminal and the neutral.
- The bounds `vm_pp_lb` and `vm_pp_ub` specify the same lower and upper bound for the magnitude of the difference of all phase terminals.
- `vm_ng_ub` specifies an upper bound for the neutral terminal, the lower bound is typically zero.

If all of these are specified, these bounds also imply valid bounds for the individual voltage magnitudes,

- $\forall$ `c` $\in$ `phases`: `vm_pn_lb - vm_ng_ub <= |v[c]| <= vm_pn_ub + vm_ng_ub`
- `0 <= |v[neutral]|<= vm_ng_ub`

Instead of defining the bounds directly, they can be specified through an associated voltage zone.

| Name           | Default   | Type        | Units | Used   | Description                                                   |
| -------------- | --------- | ----------- | ----- | ------ | ------------------------------------------------------------- |
| `phases`       | `[1,2,3]` | Vector{Any} |       | always | Identifies the terminal that represents the neutral conductor |
| `neutral`      | `4`       | `Any`       |       | always | Identifies the terminal that represents the neutral conductor |
| `voltage_zone` |           | `Any`       |       | always | id of an associated voltage zone                              |
| `vm_pn_lb`     |           | `Real`      |       | opf    | Minimum phase-to-neutral voltage magnitude for all phases     |
| `vm_pn_ub`     |           | `Real`      |       | opf    | Maximum phase-to-neutral voltage magnitude for all phases     |
| `vm_pp_lb`     |           | `Real`      |       | opf    | Minimum phase-to-phase voltage magnitude for all phases       |
| `vm_pp_ub`     |           | `Real`      |       | opf    | Maximum phase-to-phase voltage magnitude for all phases       |
| `vm_ng_ub`     |           | `Real`      |       | opf    | Maximum neutral-to-ground voltage magnitude                   |

## Edge Objects

These objects represent edges on the power grid and therefore require `f_bus` and `t_bus` (or `buses` in the case of transformers), and `f_connections` and `t_connections` (or `connections` in the case of transformers).

### Lines (`line`)

This is a pi-model branch. When a `linecode` is given, and any of `rs`, `xs`, `b_fr`, `b_to`, `g_fr` or `g_to` are specified, any of those overwrite the values on the linecode.

| Name             | Default                           | Type           | Units            | Used         | Description                                                                                                                          |
| ---------------- | --------------------------------- | -------------- | ---------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| `f_bus`          |                                   | `Any`          |                  | always       | id of from-side bus connection                                                                                                       |
| `t_bus`          |                                   | `Any`          |                  | always       | id of to-side bus connection                                                                                                         |
| `f_connections`  |                                   | `Vector{Any}`  |                  | always       | Indicates for each conductor, to which terminal of the `f_bus` it connects                                                           |
| `t_connections`  |                                   | `Vector{Any}`  |                  | always       | Indicates for each conductor, to which terminal of the `t_bus` it connects                                                           |
| `linecode`       |                                   | `Any`          |                  | always       | id of an associated linecode                                                                                                         |
| `rs`             |                                   | `Matrix{Real}` | ohm/meter        | always       | Series resistance matrix, `size=(nconductors,nconductors)`                                                                           |
| `xs`             |                                   | `Matrix{Real}` | ohm/meter        | always       | Series reactance matrix, `size=(nconductors,nconductors)`                                                                            |
| `g_fr`           | `zeros(nconductors, nconductors)` | `Matrix{Real}` | siemens/meter/Hz | always       | From-side conductance, `size=(nconductors,nconductors)`                                                                              |
| `b_fr`           | `zeros(nconductors, nconductors)` | `Matrix{Real}` | siemens/meter/Hz | always       | From-side susceptance, `size=(nconductors,nconductors)`                                                                              |
| `g_to`           | `zeros(nconductors, nconductors)` | `Matrix{Real}` | siemens/meter/Hz | always       | To-side conductance, `size=(nconductors,nconductors)`                                                                                |
| `b_to`           | `zeros(nconductors, nconductors)` | `Matrix{Real}` | siemens/meter/Hz | always       | To-side susceptance, `size=(nconductors,nconductors)`                                                                                |
| `length`         | `1.0`                             | `Real`         | meter            | always       | Length of the line                                                                                                                   |
| `cm_ub`          |                                   | `Vector{Real}` | amp              | opf          | Symmetrically applicable current rating, `size=nconductors`                                                                          |
| `sm_ub`          |                                   | `Vector{Real}` | watt             | opf          | Symmetrically applicable power rating, `size=nconductors`                                                                            |
| `pl_fr`          |                                   | `Vector{Real}` | watt             | solution     | Present active power flow on the from-side                                                                                           |
| `ql_fr`          |                                   | `Vector{Real}` | watt             | solution     | Present reactive power flow on the from-side                                                                                         |
| `pl_to`          |                                   | `Vector{Real}` | watt             | solution     | Present active power flow on the to-side                                                                                             |
| `ql_to`          |                                   | `Vector{Real}` | watt             | solution     | Present reactive power flow on the to-side                                                                                           |
| `vad_lb`         | `fill(-60.0, nphases)`            | `Vector{Real}` | degree           | opf          | Voltage angle difference lower bound                                                                                                 |
| `vad_ub`         | `fill(60.0, nphases)`             | `Vector{Real}` | degree           | opf          | Voltage angle difference upper bound                                                                                                 |
| `status`         | `1`                               | `Int`          |                  | always       | `1` or `0`. Indicates if component is enabled or disabled, respectively                                                              |
| `{}_time_series` |                                   | `Any`          |                  | multinetwork | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for `status`, `rs`, `xs`, `cm_ub`, `sm_ub` |

- which takes presidence, `cm_ub` or `sm_ub` if both are specified?

Add example for linecodes/lines off different size (conductors)

### Transformers (`transformer`)

These are n-winding (`nwinding`), n-phase (`nphase`), lossy transformers. Note that most properties are now Vectors (or Vectors of Vectors), indexed over the windings.

| Name             | Default                              | Type                   | Units | Used         | Description                                                                                                                                        |
| ---------------- | ------------------------------------ | ---------------------- | ----- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `buses`          |                                      | `Vector{Any}`          |       | always       | List of bus for each winding, `size=nwindings`                                                                                                     |
| `connections`    |                                      | `Vector{Vector{Any}}`  |       | always       | List of connection for each winding, `size=((nconductors),nwindings)`                                                                              |
| `configurations` | `fill("wye", nwindings)`             | `Vector{String}`       |       | always       | `"wye"` or `"delta"`. List of configuration for each winding, `size=nwindings`                                                                     |
| `xsc`            | `nwindings == 2 ? [0.0] : zeros(3)`  | `Vector{Real}`         | ohm   | always       | List of short-circuit reactances between each pair of windings; enter as a list of the upper-triangle elements                                     |
| `rs`             | `zeros(nwindings)`                   | `Vector{Real}`         | ohm   | always       | List of the winding resistance for each winding                                                                                                    |
| `imag`           |                                      | `Real`                 |       | always       |                                                                                                                                                    |
| `noloadloss`     |                                      | `Real`                 |       | always       |                                                                                                                                                    |
| `tm_nom`         | `ones(nwindings)`                    | `Vector{Real}`         |       | always       | Nominal tap ratio for the transformer, `size=nwindings` (multiplier)                                                                               |
| `tm_ub`          |                                      | `Vector{Vector{Real}}` |       | opf          | Maximum tap ratio for each winding and phase, `size=((nphases),nwindings)` (base=`tm_nom`)                                                         |
| `tm_lb`          |                                      | `Vector{Vector{Real}}` |       | opf          | Minimum tap ratio for for each winding and phase, `size=((nphases),nwindings)` (base=`tm_nom`)                                                     |
| `tm_set`         | `fill(fill(1.0,nphases),nwindings)`  | `Vector{Vector{Real}}` |       | always       | Set tap ratio for each winding and phase, `size=((nphases),nwindings)` (base=`tm_nom`)                                                             |
| `tm_fix`         | `fill(fill(true,nphases),nwindings)` | `Vector{Vector{Bool}}` |       | oltc         | Indicates for each winding and phase whether the tap ratio is fixed, `size=((nphases),nwindings)`                                                  |
| `polarity`       | `fill(1,nwindings)`                  | `Vector{Int}`          |       | always       |                                                                                                                                                    |
| `vnom`           |                                      | `Vector{Real}`         | volt  | always       |                                                                                                                                                    |
| `snom`           |                                      | `Vector{Real}`         | watt  | always       |                                                                                                                                                    |
| `tm`             |                                      | `Vector{Vector{Real}}` |       | solution     | Present tap ratio for each winding and phase                                                                                                       |
| `status`         | `1`                                  | `Int`                  |       | always       | `1` or `0`. Indicates if component is enabled or disabled, respectively                                                                            |
| `{}_time_series` |                                      | `Any`                  |       | multinetwork | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for `status`, `tm`, `tm_fix`, `tm_set`, `tm_ub`, `tm_lb` |

#### Assymetric, Lossless, Two-Winding (AL2W) Transformers

Special case of the Generic transformer. These are transformers are assymetric (A), lossless (L) and two-winding (2W). Assymetric refers to the fact that the secondary is always has a `wye` configuration, whilst the primary can be `delta`. The table below indicates alternate, more simple ways to specify the special case of an AL2W Transformer. `xsc` and `rs` cannot be specified for an AL2W transformer. To use this definition format, all of `f_bus`, `t_bus`, `f_connections`, `t_connections`, and `configuration` must be used, and none of `buses`, `connections`, `configurations` may be used.

| Name            | Default              | Type           | Units | Used   | Description                                                                                                 |
| --------------- | -------------------- | -------------- | ----- | ------ | ----------------------------------------------------------------------------------------------------------- |
| `f_bus`         |                      | `Any`          |       | always | Alternative way to specify `buses`, requires both `f_bus` and `t_bus`                                       |
| `t_bus`         |                      | `Any`          |       | always | Alternative  way to specify `buses`, requires both `f_bus` and `t_bus`                                      |
| `f_connections` |                      | `Vector{Any}`  |       | always | Alternative way to specify `connections`, requires both `f_connections` and `t_connections`, `size=nphases` |
| `t_connections` |                      | `Vector{Any}`  |       | always | Alternative way to specify `connections`, requires both `f_connections` and `t_connections`, `size=nphases` |
| `configuration` | `"wye"`              | `String`       |       | always | `"wye"` or `"delta"`. Alternative way to specify the from-side configuration, to-side is always `"wye"`     |
| `tm_nom`        | `1.0`                | `Real`         |       | always | Nominal tap ratio for the transformer (multiplier)                                                          |
| `tm_ub`         |                      | `Vector{Real}` |       | opf    | Maximum tap ratio for each phase (base=`tm_nom`), `size=nphases`                                            |
| `tm_lb`         |                      | `Vector{Real}` |       | opf    | Minimum tap ratio for each phase (base=`tm_nom`), `size=nphases`                                            |
| `tm_set`        | `fill(1.0,nphases)`  | `Vector{Real}` |       | always | Set tap ratio for each phase (base=`tm_nom`), `size=nphases`                                                |
| `tm_fix`        | `fill(true,nphases)` | `Vector{Real}` |       | oltc   | Indicates for each phase whether the tap ratio is fixed, `size=nphases`                                     |

### Switches (`switch`)

Switches without any of `rs`, `xs`, `g_fr`, `b_fr`, `g_to`, `b_to` or alternatively, a linecode, defined the switch will be treated as lossless. If lossy parameters are defined, `switch` objects will be decomposed into virtual `branch` & `bus`, and an ideal `switch`.

| Name             | Default                  | Type           | Units   | Used         | Description                                                                                                                                   |
| ---------------- | ------------------------ | -------------- | ------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `f_bus`          |                          | `Any`          |         | always       | id of from-side bus connection                                                                                                                |
| `t_bus`          |                          | `Any`          |         | always       | id of to-side bus connection                                                                                                                  |
| `f_connections`  |                          | `Vector{Any}`  |         | always       | Indicates for each conductor, to which terminal of the `f_bus` it connects                                                                    |
| `t_connections`  |                          | `Vector{Any}`  |         | always       | Indicates for each conductor, to which terminal of the `t_bus` it connects                                                                    |
| `cm_ub`          |                          | `Vector{Real}` | amp     | opf          | Symmetrically applicable current rating                                                                                                       |
| `sm_ub`          |                          | `Vector{Real}` | watt    | opf          | Symmetrically applicable power rating                                                                                                         |
| `linecode`       |                          | `String`       |         | always       | id of an associated linecode                                                                                                                  |
| `rs`             | `zeros(nphases,nphases)` | `Matrix{Real}` | ohm     | always       | Series resistance matrix, `size=(nphases,nphases)`                                                                                            |
| `xs`             | `zeros(nphases,nphases)` | `Matrix{Real}` | ohm     | always       | Series reactance matrix, `size=(nphases,nphases)`                                                                                             |
| `g_fr`           | `zeros(nphases,nphases)` | `Matrix{Real}` | siemens | always       | From-side conductance, `size=(nphases,nphases)`,                                                                                              |
| `b_fr`           | `zeros(nphases,nphases)` | `Matrix{Real}` | siemens | always       | From-side susceptance, `size=(nphases,nphases)`                                                                                               |
| `g_to`           | `zeros(nphases,nphases)` | `Matrix{Real}` | siemens | always       | To-side conductance, `size=(nphases,nphases)`                                                                                                 |
| `b_to`           | `zeros(nphases,nphases)` | `Matrix{Real}` | siemens | always       | To-side susceptance, `size=(nphases,nphases)`                                                                                                 |
| `psw`            |                          | `Vector{Real}` | watt    | solution     | Present value of active power flow across the switch                                                                                          |
| `qsw`            |                          | `Vector{Real}` | var     | solution     | Present value of reactive power flow across the switch                                                                                        |
| `state`          | `1`                      | `Int`          |         | always       | `1`: closed or `0`: open, to indicate state of switch                                                                                         |
| `status`         | `1`                      | `Int`          |         | always       | `1` or `0`. Indicates if component is enabled or disabled, respectively                                                                       |
| `{}_time_series` |                          | `Any`          |         | multinetwork | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for `status`, `state`, `rs`, `xs`, `cm_ub`, `sm_ub` |

#### Fuses (`fuse`)

Special Case of switch, with its own separate component category for easier tracking. Shares the fields of switch, with these additions.

| Name                    | Default | Type     | Units | Used | Description                                                             |
| ----------------------- | ------- | -------- | ----- | ---- | ----------------------------------------------------------------------- |
| `fuse_curve`            | `""`    | `String` |       |      | id of `curve` object that specifies the fuse blowing condition          |
| `minimum_melting_curve` | `""`    | `String` |       |      | id of `curve` that specifies the minimum melting conditions of the fuse |

### Line Reactors (`line_reactor`)

Line reactors are _e.g._ inductors can can be used to protect against input power distruptions

### Series Capacitors (`series_capacitor`)

Series capacitors can be used to _e.g._ compensate inductance on lines

## Node Objects

These are objects that have single bus connections. Every object will have at least `bus`, `connections`, and `status`.

### Shunts (`shunt`)

| Name             | Default | Type           | Units   | Used         | Description                                                                                                        |
| ---------------- | ------- | -------------- | ------- | ------------ | ------------------------------------------------------------------------------------------------------------------ |
| `bus`            |         | `Any`          |         | always       | id of bus connection                                                                                               |
| `connections`    |         | `Vector{Any}`  |         | always       | Ordered list of connected conductors, `size=nconductors`                                                           |
| `gs`             |         | `Matrix{Real}` | siemens | always       | Conductance, `size=(nconductors,nconductors)`                                                                      |
| `bs`             |         | `Matrix{Real}` | siemens | always       | Susceptance, `size=(nconductors,nconductors)`                                                                      |
| `status`         | `1`     | `Bool`         |         | always       | `1` or `0`. Indicates if component is enabled or disabled, respectively                                            |
| `{}_time_series` |         | `Any`          |         | multinetwork | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for `status`, `gs`, `bs` |

#### Shunt Capacitors (`shunt_capacitor`)

This is a special case of `shunt` with its own data category for easier tracking.

| Name             | Default | Type           | Units   | Used         | Description                                                                                                  |
| ---------------- | ------- | -------------- | ------- | ------------ | ------------------------------------------------------------------------------------------------------------ |
| `bus`            |         | `Any`          |         | always       | id of bus connection                                                                                         |
| `connections`    |         | `Vector{Any}`  |         | always       | Ordered list of connected conductors, `size=nconductors`                                                     |
| `configuration`  | `"wye"` | `String`       |         | always       | `"wye"` or `"delta"`. If `"wye"`, `connections[end]=neutral`. For `"delta"`, 2 or 3 connections only         |
| `bs`             |         | `Matrix{Real}` | siemens | always       | Conductance, `size=(nconductors,nconductors)`                                                                |
| `vnom`           |         | `Real`         | volt    | always       | Nominal voltage (multiplier)                                                                                 |
| `status`         | `1`     | `Bool`         |         | always       | `1` or `0`. Indicates if component is enabled or disabled, respectively                                      |
| `{}_time_series` |         | `Any`          |         | multinetwork | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for `status`, `bs` |

#### Shunt Reactors (`shunt_reactor`)

This is a special case of `shunt` with its own data category for easier tracking.

| Name             | Default | Type           | Units   | Used         | Description                                                                                                  |
| ---------------- | ------- | -------------- | ------- | ------------ | ------------------------------------------------------------------------------------------------------------ |
| `bus`            |         | `Any`          |         | always       | id of bus connection                                                                                         |
| `connections`    |         | `Vector{Any}`  |         | always       | Ordered list of connected conductors, `size=nconductors`                                                     |
| `configuration`  | `"wye"` | `String`       |         | always       | `"wye"` or `"delta"`. If `"wye"`, `connections[end]=neutral`. For `"delta"`, 2 or 3 connections only         |
| `bs`             |         | `Matrix{Real}` | siemens | always       | Conductance, `size=(nconductors,nconductors)`                                                                |
| `vnom`           |         | `Real`         | volt    | always       | Nominal voltage (multiplier)                                                                                 |
| `status`         | `1`     | `Bool`         |         | always       | `1` or `0`. Indicates if component is enabled or disabled, respectively                                      |
| `{}_time_series` |         | `Any`          |         | multinetwork | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for `status`, `bs` |

### Loads (`load`)

| Name             | Default            | Type           | Units | Used                      | Description                                                                                                                             |
| ---------------- | ------------------ | -------------- | ----- | ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `bus`            |                    | `Any`          |       | always                    | id of bus connection                                                                                                                    |
| `connections`    |                    | `Vector{Any}`  |       | always                    | Ordered list of connected conductors, `size=nconductors`                                                                                |
| `configuration`  | `"wye"`            | `String`       |       | always                    | `"wye"` or "`delta`". If `"wye"`, `connections[end]=neutral`                                                                            |
| `model`          | `"constant_power"` | `String`       |       | always                    | `"constant_power"`, `"constant_impedance"`, `"constant_current"`, `"exponential"`, or `"zip"`. Indicates the type of voltage-dependency |
| `pd_nom`         |                    | `Vector{Real}` | watt  | always                    | Nominal active load, with respect to `vnom`, `size=nphases`                                                                             |
| `qd_nom`         |                    | `Vector{Real}` | var   | always                    | Nominal reactive load, with respect to `vnom`, `size=nphases`                                                                           |
| `vnom`           |                    | `Real`         | volt  | `model!="constant_power"` | Nominal voltage (multiplier)                                                                                                            |
| `status`         | `1`                | `Bool`         |       | always                    | `1` or `0`. Indicates if component is enabled or disabled, respectively                                                                 |
| `{}_time_series` |                    | `Any`          |       | multinetwork              | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for `status`, `pd_nom`, `qd_nom`              |

Multi-phase loads define a number of individual loads connected between two terminals each. How they are connected, is defined both by `configuration` and `connections`. The table below indicates the value of `configuration` and lengths of the other properties for a consistent definition,

| `configuration` | `|connections|` | `|pd_nom|=|qd_nom|=|pd_exp|=...` |
| --------------- | --------------- | -------------------------------- |
| `delta`         | `2`             | `1`                              |
| `delta`         | `3`             | `3`                              |
| `wye`           | `2`             | `1`                              |
| `wye`           | `3`             | `2`                              |
| `wye`           | `N`             | `N-1`                            |

Note that for delta loads, only 2 and 3 connections are allowed. Each individual load `i` is connected between two terminals, exposed to a voltage magnitude `v[i]`, which leads to a consumption `pd[i]+j*qd[i]`. The `model` then defines the relationship between these quantities,

| model                | `pd[i]/pd_nom[i]=` | `qd[i]/qd_nom[i]=` |
| -------------------- | ------------------ | ------------------ |
| `constant_power`     | `1`                | `1`                |
| `constant_current`   | `(v[i]/vnom)`      | `(v[i]/vnom)`      |
| `constant_impedance` | `(v[i]/vnom)^2`    | `(v[i]/vnom)^2`    |

Two more model types are supported, which need additional fields and are defined below.

#### `model="exponential"`

- `(pd[i]/pd_nom[i]) = (v[i]/vnom)^pd_exp[i]`
- `(qd[i]/qd_nom[i]) = (v[i]/vnom)^qd_exp[i]`

| Name     | Default | Type   | Units | Used                   | Description |
| -------- | ------- | ------ | ----- | ---------------------- | ----------- |
| `pd_exp` |         | `Real` |       | `model=="exponential"` |             |
| `qd_exp` |         | `Real` |       | `model=="exponential"` |             |

#### `model="zip"`

- `(pd[i]/pd_nom) = pd_cz[i]*(v[i]/vnom)^2 + pd_ci[i]*(v[i]/vnom) + pd_cp[i]`
- `(qd[i]/qd_nom) = qd_cz[i]*(v[i]/vnom)^2 + qd_ci[i]*(v[i]/vnom) + qd_cp[i]`

| Name    | Default | Type   | Units | Used           | Description                  |
| ------- | ------- | ------ | ----- | -------------- | ---------------------------- |
| `vnom`  |         | `Real` | volt  | `model=="zip"` | Nominal voltage (multiplier) |
| `pd_cz` |         | `Real` |       | `model=="zip"` |                              |
| `pd_ci` |         | `Real` |       | `model=="zip"` |                              |
| `pd_cp` |         | `Real` |       | `model=="zip"` |                              |
| `qd_cz` |         | `Real` |       | `model=="zip"` |                              |
| `qd_ci` |         | `Real` |       | `model=="zip"` |                              |
| `qd_cp` |         | `Real` |       | `model=="zip"` |                              |

### Generators `generator` (or Synchronous Machines `synchronous_machine`?)

| Name             | Default              | Type           | Units | Used         | Description                                                                                                                                                                                                     |
| ---------------- | -------------------- | -------------- | ----- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bus`            |                      | `Any`          |       | always       | id of bus connection                                                                                                                                                                                            |
| `connections`    |                      | `Vector{Any}`  |       | always       | Ordered list of connected conductors, `size=nconductors`                                                                                                                                                        |
| `configuration`  | `"wye"`              | `String`       |       | always       | `"wye"` or `"delta"`. If `"wye"`, `connections[end]=neutral`                                                                                                                                                    |
| `control_mode`   | `1`                  | `Int`          |       | always       | Model of the generator. `1`: Constant P,Q; `2`: Constant P,\|V\|; `3`: Constant Z; `4`: Current-limited, constant P,Q (e.g. some inverters).  `1` and `2` supported, other models have plans for future support |
| `vg`             |                      | `Vector{Real}` | volt  | `model==2`   | Voltage magnitude setpoint                                                                                                                                                                                      |
| `pg_lb`          | `zeros(nphases)`     | `Vector{Real}` | watt  | opf          | Lower bound on active power generation per phase, `size=nphases`                                                                                                                                                |
| `pg_ub`          | `fill(Inf, nphases)` | `Vector{Real}` | watt  | opf          | Upper bound on active power generation per phase, `size=nphases`                                                                                                                                                |
| `qg_lb`          | `-pg_ub`             | `Vector{Real}` | var   | opf          | Lower bound on reactive power generation per phase, `size=nphases`                                                                                                                                              |
| `qg_ub`          | `pg_ub`              | `Vector{Real}` | var   | opf          | Upper bound on reactive power generation per phase, `size=nphases`                                                                                                                                              |
| `pg`             |                      | `Vector{Real}` | watt  | solution     | Present active power generation per phase, `size=nphases`                                                                                                                                                       |
| `qg`             |                      | `Vector{Real}` | var   | solution     | Present reactive power generation per phase, `size=nphases`                                                                                                                                                     |
| `status`         | `1`                  | `Bool`         |       | always       | `1` or `0`. Indicates if component is enabled or disabled, respectively                                                                                                                                         |
| `{}_time_series` |                      | `Any`          |       | multinetwork | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for `status`, `pg_lb`, `qg_lb`, `pg_ub`, `qg_ub`, `pg`, `qg`                                                          |

#### `generator` Cost Model

The generator cost model is currently specified by the following fields.

| Name                 | Default           | Type           | Units | Used         | Description                                                                                                                         |
| -------------------- | ----------------- | -------------- | ----- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| `cost_pg_model`      | `2`               | `Int`          |       | opf          | Cost model type, `1` = piecewise-linear, `2` = polynomial                                                                           |
| `cost_pg_parameters` | `[0.0, 1.0, 0.0]` | `Vector{Real}` | $/MVA | opf          | Cost model polynomial                                                                                                               |
| `{}_time_series`     |                   | `Any`          |       | multinetwork | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for `cost_pg_model`, `cost_pg_parameters` |

### Photovoltaic Systems (`solar`)

TODO Loss model, Inverter settings, Irradiance Model

| Name             | Default | Type           | Units | Used         | Description                                                                                                                                            |
| ---------------- | ------- | -------------- | ----- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `bus`            |         | `Any`          |       | always       | id of bus connection                                                                                                                                   |
| `connections`    |         | `Vector{Any}`  |       | always       | Ordered list of connected conductors, `size=nconductors`                                                                                               |
| `configuration`  | `"wye"` | `String`       |       | always       | `"wye"` or `"delta"`. If `"wye"`, `connections[end]=neutral`                                                                                           |
| `pg_lb`          |         | `Vector{Real}` | watt  | opf          | Lower bound on active power generation per phase, `size=nphases`                                                                                       |
| `pg_ub`          |         | `Vector{Real}` | watt  | opf          | Upper bound on active power generation per phase, `size=nphases`                                                                                       |
| `qg_lb`          |         | `Vector{Real}` | var   | opf          | Lower bound on reactive power generation per phase, `size=nphases`                                                                                     |
| `qg_ub`          |         | `Vector{Real}` | var   | opf          | Upper bound on reactive power generation per phase, `size=nphases`                                                                                     |
| `pg`             |         | `Vector{Real}` | watt  | solution     | Present active power generation per phase, `size=nphases`                                                                                              |
| `qg`             |         | `Vector{Real}` | var   | solution     | Present reactive power generation per phase, `size=nphases`                                                                                            |
| `status`         | `1`     | `Bool`         |       | always       | `1` or `0`. Indicates if component is enabled or disabled, respectively                                                                                |
| `{}_time_series` |         | `Any`          |       | multinetwork | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for `status`, `pg_lb`, `qg_lb`, `pg_ub`, `qg_ub`, `pg`, `qg` |

#### `solar` Cost Model

The cost model for a photovoltaic system currently matches that of generators.

| Name                 | Default           | Type           | Units | Used         | Description                                                                                                                         |
| -------------------- | ----------------- | -------------- | ----- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| `cost_pg_model`      | `2`               | `Int`          |       | opf          | Cost model type, `1` = piecewise-linear, `2` = polynomial                                                                           |
| `cost_pg_parameters` | `[0.0, 1.0, 0.0]` | `Vector{Real}` | $/MVA | opf          | Cost model polynomial                                                                                                               |
| `{}_time_series`     |                   | `Any`          |       | multinetwork | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for `cost_pg_model`, `cost_pg_parameters` |

### Wind Turbine Systems (`wind`)

Wind turbine systems are most closely approximated by induction machines, also known as asynchornous machines. These are not currently supported, but there is plans to support them in the future.

### Storage (`storage`)

A storage object is a flexible component that can represent a variety of energy storage objects, like Li-ion batteries, hydrogen fuel cells, flywheels, etc.

- How to include the inverter model for this? Similar issue as for a PV generator

| Name                   | Default | Type           | Units   | Used         | Description                                                                                                                                                                                                                             |
| ---------------------- | ------- | -------------- | ------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bus`                  |         | `Any`          |         | always       | id of bus connection                                                                                                                                                                                                                    |
| `connections`          |         | `Vector{Any}`  |         | always       | Ordered list of connected conductors, `size=nconductors`                                                                                                                                                                                |
| `configuration`        | `"wye"` | `String`       |         | always       | `"wye"` or `"delta"`. If `"wye"`, `connections[end]=neutral`                                                                                                                                                                            |
| `energy`               |         | `Real`         | watt-hr | always       | Stored energy                                                                                                                                                                                                                           |
| `energy_ub`            |         | `Real`         |         | opf          | maximum energy rating                                                                                                                                                                                                                   |
| `charge_ub`            |         | `Real`         |         | opf          | maximum charge rating                                                                                                                                                                                                                   |
| `discharge_ub`         |         | `Real`         |         | opf          | maximum discharge rating                                                                                                                                                                                                                |
| `sm_ub`                |         | `Vector{Real}` | watt    | opf          | Power rating, `size=nphases`                                                                                                                                                                                                            |
| `cm_ub`                |         | `Vector{Real}` | amp     | opf          | Current rating, `size=nphases`                                                                                                                                                                                                          |
| `charge_efficiency`    |         | `Real`         | percent | always       | charging efficiency (losses)                                                                                                                                                                                                            |
| `discharge_efficiency` |         | `Real`         | percent | always       | disharging efficiency (losses)                                                                                                                                                                                                          |
| `qs_ub`                |         | `Vector{Real}` |         | opf          | Maximum reactive power injection, `size=nphases`                                                                                                                                                                                        |
| `qs_lb`                |         | `Vector{Real}` |         | opf          | Minimum reactive power injection, `size=nphases`                                                                                                                                                                                        |
| `rs`                   |         | `Vector{Real}` | ohm     | always       | converter resistance                                                                                                                                                                                                                    |
| `xs`                   |         | `Vector{Real}` | ohm     | always       | converter reactance                                                                                                                                                                                                                     |
| `pex`                  |         | `Real`         |         | always       | Total active power standby exogenous flow (loss)                                                                                                                                                                                        |
| `qex`                  |         | `Real`         |         | always       | Total reactive power standby exogenous flow (loss)                                                                                                                                                                                      |
| `ps`                   |         | `Vector{Real}` | watt    | solution     | Present active power injection                                                                                                                                                                                                          |
| `qs`                   |         | `Vector{Real}` | var     | solution     | Present reactive power injection                                                                                                                                                                                                        |
| `status`               | `1`     | `Bool`         |         | opf          | `1` or `0`. Indicates if component is enabled or disabled, respectively                                                                                                                                                                 |
| `{}_time_series`       |         | `Any`          |         | multinetwork | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for `status`, `energy`, `energy_ub`, `charge_ub`, `discharge_ub`, `charge_efficiency`, `discharge_efficiency`, `qs_ub`, `qs_lb`, `pex`, `qex` |

### Voltage Sources (`voltage_source`)

A voltage source is a source of power at a set voltage magnitude and angle connected to a slack bus. If `rs` or `xs` are not specified, the voltage source is assumed to be lossless, otherwise virtual `branch` and `bus` will be created in the mathematical model to represent the internal losses of the voltage source.

| Name             | Default                          | Type           | Units  | Used         | Description                                                                                                        |
| ---------------- | -------------------------------- | -------------- | ------ | ------------ | ------------------------------------------------------------------------------------------------------------------ |
| `bus`            |                                  | `Any`          |        | always       | id of bus connection                                                                                               |
| `connections`    |                                  | `Vector{Any}`  |        | always       | Ordered list of connected conductors, `size=nconductors`                                                           |
| `configuration`  | `"wye"`                          | `String`       |        | always       | `"wye"` or `"delta"`. If `"wye"`, `connections[end]=neutral`                                                       |
| `vm`             | `ones(nphases)`                  | `Vector{Real}` | volt   | always       | Voltage magnitude set at slack bus, `size=nphases`                                                                 |
| `va`             | `0.0`                            | `Real`         | degree | always       | Voltage angle offset at slack bus, applies symmetrically to each phase angle                                       |
| `rs`             | `zeros(nconductors,nconductors)` | `Matrix{Real}` | ohm    | always       | Internal series resistance of voltage source, `size=(nconductors,nconductors)`                                     |
| `xs`             | `zeros(nconductors,nconductors)` | `Matrix{Real}` | ohm    | always       | Internal series reactance of voltage soure, `size=(nconductors,nconductors)`                                       |
| `status`         | `1`                              | `Bool`         |        | always       | `1` or `0`. Indicates if component is enabled or disabled, respectively                                            |
| `{}_time_series` |                                  | `Any`          |        | multinetwork | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for `status`, `vm`, `va` |

## Data Objects (codes, curves, shapes)

These objects are referenced by node and edge objects, but are not part of the network themselves, only containing data.

### Linecodes (`linecode`)

Linecodes are easy ways to specify properties common to multiple lines.

- Should the linecode also include a `cm_ub` and `sm_ub`? I think not. `cm_ub` yes, `sm_ub` does not make sense. `sm_ub`, if desired, should be specified at the line level
- Does `cmatrix` makes more sense than `g_fr, b_fr, g_to, b_to` when parametrized per length? No, enforce symmetry. If needed place a shunt or inject at low-level.

| Name             | Default                          | Type           | Units            | Used         | Description                                                                                                       |
| ---------------- | -------------------------------- | -------------- | ---------------- | ------------ | ----------------------------------------------------------------------------------------------------------------- |
| `rs`             |                                  | `Matrix{Real}` | ohm/meter        | always       | Series resistance, `size=(nconductors,nconductors)`                                                               |
| `xs`             |                                  | `Matrix{Real}` | ohm/meter        | always       | Series reactance, `size=(nconductors,nconductors)`                                                                |
| `g_fr`           | `zeros(nconductors,nconductors)` | `Matrix{Real}` | siemens/meter/Hz | always       | From-side conductance, `size=(nconductors,nconductors)`                                                           |
| `b_fr`           | `zeros(nconductors,nconductors)` | `Matrix{Real}` | siemens/meter/Hz | always       | From-side susceptance, `size=(nconductors,nconductors)`                                                           |
| `g_to`           | `zeros(nconductors,nconductors)` | `Matrix{Real}` | siemens/meter/Hz | always       | To-side conductance, `size=(nconductors,nconductors)`                                                             |
| `b_to`           | `zeros(nconductors,nconductors)` | `Matrix{Real}` | siemens/meter/Hz | always       | To-side susceptance, `size=(nconductors,nconductors)`                                                             |
| `cm_ub`          |                                  | `Vector{Real}` | ampere           | always       | maximum current per conductor, symmetrically applicable                                                           |
| `{}_time_series` |                                  | `Any`          |                  | multinetwork | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for `rs`, `xs`, `cm_ub` |

### Transformer Codes (`xfmrcode`)

Transformer codes are easy ways to specify properties common to multiple transformers

| Name             | Default                                | Type                   | Units | Used         | Description                                                                                                                                               |
| ---------------- | -------------------------------------- | ---------------------- | ----- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `configurations` | `fill("wye", nwindings)`               | `Vector{String}`       |       | always       | `"wye"` or `"delta"`. List of configuration for each winding, `size=nwindings`                                                                            |
| `xsc`            | `[0.0]`                                | `Vector{Real}`         | ohm   | always       | List of short-circuit reactances between each pair of windings; enter as a list of the upper-triangle elements, `size=(nwindings == 2 ? 1 : 3)`           |
| `rs`             | `zeros(nwindings)`                     | `Vector{Real}`         | ohm   | always       | List of the winding resistance for each winding, `size=nwindings`                                                                                         |
| `tm_nom`         | `ones(nwindings)`                      | `Vector{Real}`         |       | always       | Nominal tap ratio for the transformer, `size=nwindings` (multiplier)                                                                                      |
| `tm_ub`          |                                        | `Vector{Vector{Real}}` |       | opf          | Maximum tap ratio for each winding and phase, `size=((nphases), nwindings)` (base=`tm_nom`)                                                               |
| `tm_lb`          |                                        | `Vector{Vector{Real}}` |       | opf          | Minimum tap ratio for for each winding and phase, `size=((nphases), nwindings)` (base=`tm_nom`)                                                           |
| `tm_set`         | `fill(fill(1.0, nphases), nwindings)`  | `Vector{Vector{Real}}` |       | always       | Set tap ratio for each winding and phase, `size=((nphases), nwindings)` (base=`tm_nom`)                                                                   |
| `tm_fix`         | `fill(fill(true, nphases), nwindings)` | `Vector{Vector{Bool}}` |       | always       | Indicates for each winding and phase whether the tap ratio is fixed, `size=((nphases), nwindings)`                                                        |
| `{}_time_series` |                                        | `Any`                  |       | multinetwork | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for `rs`, `xsc`, `tm_nom`, `tm_ub`, `tm_lb`, `tm_set`, `tm_fix` |

### Voltage Zones (`voltage_zone`)

Voltage zones are a convenient way to specify voltage bounds for a set of three-phase buses (ones with `phases`/`neutral` properties).

| Name       | Default | Type   | Units | Used   | Description                                               |
| ---------- | ------- | ------ | ----- | ------ | --------------------------------------------------------- |
| `vnom`     |         | `Real` |       | always | Nominal phase-to-neutral voltage                          |
| `vm_pn_lb` |         | `Real` |       | opf    | Minimum phase-to-neutral voltage magnitude for all phases |
| `vm_pn_ub` |         | `Real` |       | opf    | Maximum phase-to-neutral voltage magnitude for all phases |
| `vm_pp_lb` |         | `Real` |       | opf    | Minimum phase-to-phase voltage magnitude for all phases   |
| `vm_pp_ub` |         | `Real` |       | opf    | Maximum phase-to-phase voltage magnitude for all phases   |
| `vm_ng_ub` |         | `Real` |       | opf    | Maximum neutral-to-ground voltage magnitude               |

### Curves (`curve`)

Curve objects are functions f(x) that return single values for a given x. This is useful for several objects like `solar` power-temperature curves, or efficiency curves on various objects.

| Name    | Default | Type     | Units | Used   | Description                  |
| ------- | ------- | -------- | ----- | ------ | ---------------------------- |
| `curve` |         | Function |       | always | Interpolated function f(x)=y |

### Time Series (`time_series`)

Time series objects are used to specify time series for _e.g._ load or generation forecasts.

Some parameters for components specified in this document can support a time series by appending `_time_series` to the parameter name. For example, for a `load`, if `pd_nom` is supposed to be a time series, the user would specify `"pd_nom_time_series" => time_series_id` where `time_series_id` is the `id` of an object in `time_series`, and has type `Any`. The parameters that support time series will be specified in their respective sections above.

| Name      | Default | Type           | Units | Used   | Description                                                                           |
| --------- | ------- | -------------- | ----- | ------ | ------------------------------------------------------------------------------------- |
| `time`    |         | `Vector{Real}` | hour  | always | Time points at which values are specified                                             |
| `values`  |         | `Vector{Real}` |       | always | Multipers at each time step given in `time`                                           |
| `replace` | `false` | `Bool`         |       | always | Indicates to replace with data, instead of multiply. Will only work on non-Array data |
