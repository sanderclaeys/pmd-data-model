# Data model

This document describes the `"engineering"` data model type in PowerModelsDistribution, which is transformed at runtime, or at the user's direction into a `"mathematical"` data model for optimization.

In this document,

- `nphases` refers to the number of non-neutral, non-ground active phases connected to a component,
- `nconductors` refers to all active conductors connected to a component, and
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

TODO: Specify units on each component

## Root-Level Properties

At the root level of the data structure, the following fields can be found.

| Name         | Default         | Type                 | Required | Description                                                                                                                |
| ------------ | --------------- | -------------------- | -------- | -------------------------------------------------------------------------------------------------------------------------- |
| `name`       |                 | `String`             |          | Case name                                                                                                                  |
| `data_model` | `"engineering"` | `String`             | always   | `"engineering"`, `"mathematical"`, or `"dss"`. Type of the data model (this document describes `data_model="engineering"`) |
| `settings`   | `Dict()`        | `Dict{String,<:Any}` | always   | Base settings for the data model, see Settings section below for details                                                   |

## Settings (`settings`)

At the root-level of the data model a `settings` dictionary object is expected, containing the following fields.

| Name             | Default | Type   | Units | Required | Description                                                    |
| ---------------- | ------- | ------ | ----- | -------- | -------------------------------------------------------------- |
| `v_var_scalar`   | `1e3`   | `Real` |       | always   | Scalar for voltage values                                      |
| `vbase`          |         | `Real` |       | always   | Voltage base (_i.e._ basekv) at `base_bus`                     |
| `sbase`          |         | `Real` |       | always   | Power base (baseMVA) at `base_bus`                             |
| `base_bus`       |         | `Any`  |       | always   | id of bus at which `vbase` and `sbase` apply                   |
| `base_frequency` | `60.0`  | `Real` | Hz    | always   | Frequency base, _i.e._ the base frequency of the whole circuit |

## Buses (`bus`)

The data model below allows us to include buses of arbitrary many terminals (_i.e._, more than the usual four). This would be useful for

- underground lines with multiple neutrals which are not joined at every bus;
- distribution lines that carry several conventional lines in parallel (see for example the quad circuits in NEVTestCase).

| Name             | Default     | Type            | Units  | Required | Description                                                                                                 |
| ---------------- | ----------- | --------------- | ------ | -------- | ----------------------------------------------------------------------------------------------------------- |
| `terminals`      | `[1,2,3,4]` | `Vector{Any}`   |        | always   | Terminals for which the bus has active connections                                                          |
| `vm_lb`          |             | `Vector{Real}`  |        | opf      | Minimum conductor-to-ground voltage magnitude, `size=nphases`                                               |
| `vm_ub`          |             | `Vector{Real}`  |        | opf      | Maximum conductor-to-ground voltage magnitude, `size=nphases`                                               |
| `vm_pair_ub`     |             | `Vector{Tuple}` |        | opf      | _e.g._  `[(1,2,210)]` means \|U1-U2\|>210                                                                   |
| `vm_pair_lb`     |             | `Vector{Tuple}` |        | opf      | _e.g._  `[(1,2,230)]` means \|U1-U2\|<230                                                                   |
| `grounded`       | `[]`        | `Vector{Any}`   |        |          | List of terminals which are grounded                                                                        |
| `rg`             | `[]`        | `Vector{Real}`  |        |          | Resistance of each defined grounding, `size=length(grounded)`                                               |
| `xg`             | `[]`        | `Vector{Real}`  |        |          | Reactance of each defined grounding, `size=length(grounded)`                                                |
| `vm`             |             | `Vector{Real}`  | volt   |          | Voltage magnitude at bus                                                                                    |
| `va`             |             | `Vector{Real}`  | degree |          | Voltage angle at bus                                                                                        |
| `status`         | `1`         | `Bool`          |        | always   | `1` or `0`. Indicates if component is enabled or disabled, respectively                                     |
| `{}_time_series` |             | `Any`           |        |          | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for any parameter |

The tricky part is how to encode bounds for these type of buses. The most general is defining a list of three-tuples. Take for example a typical bus in a three-phase, four-wire network, where `terminals=[a,b,c,n]`. Such a bus might have

- phase-to-neutral bounds `vm_pn_ub=250`, `vm_pn_lb=210`
- and phase-to-phase bounds `vm_pp_ub=440`, `vm_pp_lb=360`.

We can then define this equivalently as

- `vm_pair_ub = [(a,n,250), (b,n,250), (c,n,250), (a,b,440), (b,c,440), (c,a,440)]`
- `vm_pair_lb = [(a,n,210), (b,n,210), (c,n,210), (a,b,360), (b,c,360), (c,a,360)]`

If terminal `4` is grounding through an impedance `Z=1+j2`, we write

- `grounded=[4]`, `rg=[1]`, `xg=[2]`

Since this might be confusing for novice users, we also allow the user to define bounds through the following component.

### Special Case: three-phase bus

- Specify bounds relative to some `vnom`? Yes, but in voltage_zones

Much of the difficulty arrises from supporting phase-to-phase bounds for both two-phase and three-phase systems. For example, a 2-phase `[a,b]` system only has 1 phase-to-phase bound, whilst a 3-phase system `[a,b,c]` has 3!

A 4-phase system `[a,b,c,d]` is not well-defined; should there be phase-to-phase bounds for each permutation, _i.e._, `[(a,b), (a,c), (a,d), (b,c), (b,d), (c,d)]`, or only for adjacent ones as it would apply to a 4-phase delta connection, `[(a,b), (b,c), (c,d), (a,d)]`? For three-phase systems these two options coincide.

In order to avoid all of this confusion, we introduce this component, which is at most 3-phase, `|phases|<=3`, and which specifies all bounds symmetrically. For example,

- `phases=[1,2,3], vm_pp_ub=440` implies |U1-U2|, |U2-U3|, |U1-U3| <= 440
- `phases=[1,2], vm_pn_ub=440` implies |U1-U2| <= 440

This keeps the user from specifying things that do not make sense. This type of bus would suffice for most of the use cases.

Instead of defining the bounds directly, they can be specified through an associated voltagezone.

| Name          | Default   | Type        | Units | Required | Description                                                   |
| ------------- | --------- | ----------- | ----- | -------- | ------------------------------------------------------------- |
| `phases`      | `[1,2,3]` | Vector{Any} |       |          | Identifies the terminal that represents the neutral conductor |
| `neutral`     | `4`       | `Any`       |       |          | Identifies the terminal that represents the neutral conductor |
| `voltagezone` |           | `Any`       |       |          | id of an associated voltage zone                              |
| `vm_pn_lb`    |           | `Real`      |       |          | Minimum phase-to-neutral voltage magnitude for all phases     |
| `vm_pn_ub`    |           | `Real`      |       |          | Maximum phase-to-neutral voltage magnitude for all phases     |
| `vm_pp_lb`    |           | `Real`      |       |          | Minimum phase-to-phase voltage magnitude for all phases       |
| `vm_pp_ub`    |           | `Real`      |       |          | Maximum phase-to-phase voltage magnitude for all phases       |
| `vm_ng_ub`    |           | `Real`      |       |          | Maximum neutral-to-ground voltage magnitude                   |

## Edge Objects

These objects represent edges on the power grid and therefore require `f_bus` and `t_bus` (or `buses` in the case of transformers), and `f_connections` and `t_connections` (or `connections` in the case of transformers).

### Lines (`line`)

This is a pi-model branch. When a `linecode` is given, and any of `rs`, `xs`, `b_fr`, `b_to`, `g_fr` or `g_to` are specified, any of those overwrite the values on the linecode.

| Name             | Default | Type           | Units            | Required     | Description                                                                                                 |
| ---------------- | ------- | -------------- | ---------------- | ------------ | ----------------------------------------------------------------------------------------------------------- |
| `f_bus`          |         | `Any`          |                  | always       | id of from-side bus connection                                                                              |
| `t_bus`          |         | `Any`          |                  | always       | id of to-side bus connection                                                                                |
| `f_connections`  |         | `Vector{Any}`  |                  | always       | Indicates for each conductor, to which terminal of the `f_bus` it connects                                  |
| `t_connections`  |         | `Vector{Any}`  |                  | always       | Indicates for each conductor, to which terminal of the `t_bus` it connects                                  |
| `linecode`       |         | `Any`          |                  | `!rs && !xs` | id of an associated linecode                                                                                |
| `rs`             |         | `Matrix{Real}` | ohm/meter        | `!linecode`  | Series resistance matrix, `size=(nconductors,nconductors)`                                                  |
| `xs`             |         | `Matrix{Real}` | ohm/meter        | `!linecode`  | Series reactance matrix, `size=(nconductors,nconductors)`                                                   |
| `g_fr`           |         | `Matrix{Real}` | siemens/meter/Hz |              | From-side conductance, `size=(nconductors,nconductors)`                                                     |
| `b_fr`           |         | `Matrix{Real}` | siemens/meter/Hz |              | From-side susceptance, `size=(nconductors,nconductors)`                                                     |
| `g_to`           |         | `Matrix{Real}` | siemens/meter/Hz |              | To-side conductance, `size=(nconductors,nconductors)`                                                       |
| `b_to`           |         | `Matrix{Real}` | siemens/meter/Hz |              | To-side susceptance, `size=(nconductors,nconductors)`                                                       |
| `length`         | `1.0`   | `Real`         | meter            |              | Length of the line                                                                                          |
| `cm_ub`          |         | `Vector{Real}` | amp              | opf          | Symmetrically applicable current rating, `size=nconductors`                                                 |
| `sm_ub`          |         | `Vector{Real}` | watt             | opf          | Symmetrically applicable power rating, `size=nconductors`                                                   |
| `pl_fr`          |         | `Vector{Real}` | watt             |              | Present active power flow on the from-side                                                                  |
| `ql_fr`          |         | `Vector{Real}` | watt             |              | Present reactive power flow on the from-side                                                                |
| `pl_to`          |         | `Vector{Real}` | watt             |              | Present active power flow on the to-side                                                                    |
| `ql_to`          |         | `Vector{Real}` | watt             |              | Present reactive power flow on the to-side                                                                  |
| `status`         | `1`     | `Bool`         |                  | always       | `1` or `0`. Indicates if component is enabled or disabled, respectively                                     |
| `{}_time_series` |         | `Any`          |                  |              | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for any parameter |

- which takes presidence, `cm_ub` or `sm_ub` if both are specified?

Add example for linecodes/lines off different size (conductors)

### Transformers (`transformer`)

These are n-winding (`nwinding`), n-phase (`nphase`), lossy transformers. Note that most properties are now Vectors (or Vectors of Vectors), indexed over the windings.

| Name             | Default                                | Type                   | Units | Required                           | Description                                                                                                    |
| ---------------- | -------------------------------------- | ---------------------- | ----- | ---------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `buses`          |                                        | `Vector{Any}`          |       | `!f_bus && !t_bus`                 | List of bus for each winding, `size=nwindings`                                                                 |
| `connections`    |                                        | `Vector{Vector{Any}}`  |       | `!f_connections && !t_connections` | List of connection for each winding, `size=((nconductors), nwindings)`                                         |
| `configurations` | `fill("wye", nwindings)`               | `Vector{String}`       |       | always                             | `"wye"` or `"delta"`. List of configuration for each winding, `size=nwindings`                                 |
| `xsc`            | `[0.0]`                                | `Vector{Real}`         | ohm   |                                    | List of short-circuit reactances between each pair of windings; enter as a list of the upper-triangle elements |
| `rs`             | `zeros(nwindings)`                     | `Vector{Real}`         | ohm   |                                    | List of the winding resistance for each winding                                                                |
| `tm_nom`         | `ones(nwindings)`                      | `Vector{Real}`         |       |                                    | Nominal tap ratio for the transformer, `size=nwindings` (multiplier)                                           |
| `tm_ub`          |                                        | `Vector{Vector{Real}}` |       | opf                                | Maximum tap ratio for each winding and phase, `size=((nphases), nwindings)` (base=`tm_nom`)                    |
| `tm_lb`          |                                        | `Vector{Vector{Real}}` |       | opf                                | Minimum tap ratio for for each winding and phase, `size=((nphases), nwindings)` (base=`tm_nom`)                |
| `tm_set`         | `fill(fill(1.0, nphases), nwindings)`  | `Vector{Vector{Real}}` |       |                                    | Set tap ratio for each winding and phase, `size=((nphases), nwindings)` (base=`tm_nom`)                        |
| `tm_fix`         | `fill(fill(true, nphases), nwindings)` | `Vector{Vector{Bool}}` |       |                                    | Indicates for each winding and phase whether the tap ratio is fixed, `size=((nphases), nwindings)`             |
| `status`         | `1`                                    | `Bool`                 |       | always                             | `1` or `0`. Indicates if component is enabled or disabled, respectively                                        |
| `{}_time_series` |                                        | `Any`                  |       |                                    | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for any parameter    |

#### Assymetric, Lossless, Two-Winding (AL2W) Transformers

Special case of the Generic transformer. These are transformers are assymetric (A), lossless (L) and two-winding (2W). Assymetric refers to the fact that the secondary is always has a `wye` configuration, whilst the primary can be `delta`. The table below indicates alternate, more simple ways to specify the special case of an AL2W Transformer. `xsc` and `rs` cannot be specified for an AL2W transformer.

| Name             | Default               | Type           | Units | Required       | Description                                                                                                 |
| ---------------- | --------------------- | -------------- | ----- | -------------- | ----------------------------------------------------------------------------------------------------------- |
| `f_bus`          |                       | `Any`          |       | `!buses`       | Alternative way to specify `buses`, requires both `f_bus` and `t_bus`                                       |
| `t_bus`          |                       | `Any`          |       | `!buses`       | Alternative  way to specify `buses`, requires both `f_bus` and `t_bus`                                      |
| `f_connections`  |                       | `Vector{Any}`  |       | `!connections` | Alternative way to specify `connections`, requires both `f_connections` and `t_connections`                 |
| `t_connections`  |                       | `Vector{Any}`  |       | `!connections` | Alternative way to specify `connections`, requires both `f_connections` and `t_connections`                 |
| `configuration`  | `"wye"`               | `String`       |       |                | `"wye"` or `"delta"`. Alternative way to specify the from-side configuration, to-side is always `"wye"`     |
| `tm_nom`         | `1.0`                 | `Real`         |       |                | Nominal tap ratio for the transformer (multiplier)                                                          |
| `tm_ub`          |                       | `Vector{Real}` |       | opf            | Maximum tap ratio for each phase (base=`tm_nom`)                                                            |
| `tm_lb`          |                       | `Vector{Real}` |       | opf            | Minimum tap ratio for each phase (base=`tm_nom`)                                                            |
| `tm_set`         | `fill(1.0, nphases)`  | `Vector{Real}` |       |                | Set tap ratio for each phase (base=`tm_nom`)                                                                |
| `tm_fix`         | `fill(true, nphases)` | `Vector{Real}` |       |                | Indicates for each phase whether the tap ratio is fixed                                                     |
| `{}_time_series` |                       | `Any`          |       |                | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for any parameter |

### Switches (`switch`)

Switches without any of `rs`, `xs`, `g_fr`, `b_fr`, `g_to`, `b_to` defined are assumed to be lossless. If lossy parameters are defined, `switch` objects will be decomposed into virtual `branch` & `bus`, and an ideal `switch`.

| Name             | Default | Type           | Units   | Required | Description                                                                                                 |
| ---------------- | ------- | -------------- | ------- | -------- | ----------------------------------------------------------------------------------------------------------- |
| `f_bus`          |         | `Any`          |         | always   | id of from-side bus connection                                                                              |
| `t_bus`          |         | `Any`          |         | always   | id of to-side bus connection                                                                                |
| `f_connections`  |         | `Vector{Any}`  |         | always   | Indicates for each conductor, to which terminal of the `f_bus` it connects                                  |
| `t_connections`  |         | `Vector{Any}`  |         | always   | Indicates for each conductor, to which terminal of the `t_bus` it connects                                  |
| `cm_ub`          |         | `Vector{Real}` | amp     | opf      | Symmetrically applicable current rating                                                                     |
| `sm_ub`          |         | `Vector{Real}` | watt    | opf      | Symmetrically applicable power rating                                                                       |
| `rs`             |         | `Matrix{Real}` | ohm     |          | Series resistance matrix, `size=(nphases,nphases)`                                                          |
| `xs`             |         | `Matrix{Real}` | ohm     |          | Series reactance matrix, `size=(nphases,nphases)`                                                           |
| `g_fr`           |         | `Matrix{Real}` | siemens |          | From-side conductance, `size=(nphases,nphases)`,                                                            |
| `b_fr`           |         | `Matrix{Real}` | siemens |          | From-side susceptance, `size=(nphases,nphases)`                                                             |
| `g_to`           |         | `Matrix{Real}` | siemens |          | To-side conductance, `size=(nphases,nphases)`                                                               |
| `b_to`           |         | `Matrix{Real}` | siemens |          | To-side susceptance, `size=(nphases,nphases)`                                                               |
| `psw`            |         | `Vector{Real}` | watt    |          | Present value of active power flow across the switch                                                        |
| `qsw`            |         | `Vector{Real}` | var     |          | Present value of reactive power flow across the switch                                                      |
| `state`          | `1`     | `Int`          |         | always   | `1`: closed or `0`: open, to indicate state of switch                                                       |
| `status`         | `1`     | `Bool`         |         | always   | `1` or `0`. Indicates if component is enabled or disabled, respectively                                     |
| `{}_time_series` |         | `Any`          |         |          | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for any parameter |

#### Fuses (`fuse`)

Special Case of switch, with its own separate component category for easier tracking. Shares the fields of switch, with these additions.

| Name                    | Default | Type     | Units | Required | Description                                                             |
| ----------------------- | ------- | -------- | ----- | -------- | ----------------------------------------------------------------------- |
| `fuse_curve`            |         | `String` |       |          | id of `curve` object that specifies the fuse blowing condition          |
| `minimum_melting_curve` |         | `String` |       |          | id of `curve` that specifies the minimum melting conditions of the fuse |

### Line Reactors (`line_reactor`)

TODO

### Series Capacitors (`series_capacitor`)

TODO

## Node Objects

These are objects that have single bus connections. Every object will have at least `bus`, `connections`, and `status`.

### Shunts (`shunt`)

| Name             | Default | Type           | Units   | Required | Description                                                                                                 |
| ---------------- | ------- | -------------- | ------- | -------- | ----------------------------------------------------------------------------------------------------------- |
| `bus`            |         | `Any`          |         | always   | id of bus connection                                                                                        |
| `connections`    |         | `Vector{Any}`  |         | always   | Ordered list of connected conductors                                                                        |
| `configuration`  | `"wye"` | `String`       |         | always   | `"wye"` or `"delta"`. If `"wye"`, `connections[end]=neutral`                                                |
| `gs`             |         | `Matrix{Real}` | siemens | always   | Conductance, `size=(nphases,nphases)`                                                                       |
| `bs`             |         | `Matrix{Real}` | siemens | always   | Susceptance, `size=(nphases,nphases)`                                                                       |
| `status`         | `1`     | `Bool`         |         | always   | `1` or `0`. Indicates if component is enabled or disabled, respectively                                     |
| `{}_time_series` |         | `Any`          |         |          | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for any parameter |

#### Shunt Capacitors (`shunt_capacitor`)

This is a special case of `shunt` with its own data category for easier tracking.

| Name             | Default | Type           | Units   | Required | Description                                                                                                 |
| ---------------- | ------- | -------------- | ------- | -------- | ----------------------------------------------------------------------------------------------------------- |
| `bus`            |         | `Any`          |         | always   | id of bus connection                                                                                        |
| `connections`    |         | `Vector{Any}`  |         | always   | Ordered list of connected conductors, `size=nconductors`                                                    |
| `configuration`  | `"wye"` | `String`       |         | always   | `"wye"` or `"delta"`. If `"wye"`, `connections[end]=neutral`. For `"delta"`, 2 or 3 connections only        |
| `bs`             |         | `Vector{Real}` | siemens | always   | Conductance, `size=(nconductors,nconductors)`                                                               |
| `vnom`           |         | `Real`         | volt    | always   | Nominal voltage (multiplier)                                                                                |
| `{}_time_series` |         | `Any`          |         |          | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for any parameter |

#### Shunt Reactors (`shunt_reactor`)

This is a special case of `shunt` with its own data category for easier tracking.

| Name             | Default | Type           | Units   | Required | Description                                                                                                 |
| ---------------- | ------- | -------------- | ------- | -------- | ----------------------------------------------------------------------------------------------------------- |
| `bus`            |         | `Any`          |         | always   | id of bus connection                                                                                        |
| `connections`    |         | `Vector{Any}`  |         | always   | Ordered list of connected conductors, `size=nconductors`                                                    |
| `configuration`  | `"wye"` | `String`       |         | always   | `"wye"` or `"delta"`. If `"wye"`, `connections[end]=neutral`. For `"delta"`, 2 or 3 connections only        |
| `bs`             |         | `Vector{Real}` | siemens | always   | Conductance, `size=(nconductors,nconductors)`                                                               |
| `vnom`           |         | `Real`         | volt    | always   | Nominal voltage (multiplier)                                                                                |
| `{}_time_series` |         | `Any`          |         |          | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for any parameter |

### Loads (`load`)

| Name             | Default            | Type           | Units | Required | Description                                                                                                                             |
| ---------------- | ------------------ | -------------- | ----- | -------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `bus`            |                    | `Any`          |       | always   | id of bus connection                                                                                                                    |
| `connections`    |                    | `Vector{Any}`  |       | always   | Ordered list of connected conductors                                                                                                    |
| `configuration`  | `"wye"`            | `String`       |       | always   | `"wye"` or "`delta`". If `"wye"`, `connections[end]=neutral`                                                                            |
| `model`          | `"constant_power"` | `String`       |       | always   | `"constant_power"`, `"constant_impedance"`, `"constant_current"`, `"exponential"`, or `"zip"`. Indicates the type of voltage-dependency |
| `pd_nom`         |                    | `Vector{Real}` | watt  | always   | Nominal active load, with respect to `vnom`, `size=nphases`                                                                             |
| `qd_nom`         |                    | `Vector{Real}` | var   | always   | Nominal reactive load, with respect to `vnom`, `size=nphases`                                                                           |
| `vnom`           |                    | `Real`         | volt  | always   | Nominal voltage (multiplier)                                                                                                            |
| `status`         | `1`                | `Bool`         |       | always   | `1` or `0`. Indicates if component is enabled or disabled, respectively                                                                 |
| `{}_time_series` |                    | `Any`          |       |          | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for any parameter                             |

#### `model="exponential"`

| Name     | Default | Type   | Units | Required               | Description |
| -------- | ------- | ------ | ----- | ---------------------- | ----------- |
| `pd_exp` |         | `Real` |       | `model=="exponential"` |             |
| `qd_exp` |         | `Real` |       | `model=="exponential"` |             |

#### `model="zip"`

| Name       | Default | Type   | Units | Required       | Description |
| ---------- | ------- | ------ | ----- | -------------- | ----------- |
| `pd_nom_z` |         | `Real` |       | `model=="zip"` |             |
| `pd_nom_i` |         | `Real` |       | `model=="zip"` |             |
| `pd_nom_p` |         | `Real` |       | `model=="zip"` |             |
| `qd_nom_z` |         | `Real` |       | `model=="zip"` |             |
| `qd_nom_i` |         | `Real` |       | `model=="zip"` |             |
| `qd_nom_p` |         | `Real` |       | `model=="zip"` |             |

### Generators `generator` (or Synchronous Machines `synchronous_machine`?)

| Name             | Default | Type           | Units | Required | Description                                                                                                                                                                                                     |
| ---------------- | ------- | -------------- | ----- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bus`            |         | `Any`          |       | always   | id of bus connection                                                                                                                                                                                            |
| `connections`    |         | `Vector{Any}`  |       | always   | Ordered list of connected conductors                                                                                                                                                                            |
| `configuration`  | `"wye"` | `String`       |       | always   | `"wye"` or `"delta"`. If `"wye"`, `connections[end]=neutral`                                                                                                                                                    |
| `model`          | `1`     | `Int`          |       | always   | Model of the generator. `1`: Constant P,Q; `2`: Constant P,\|V\|; `3`: Constant Z; `4`: Current-limited, constant P,Q (e.g. some inverters).  `1` and `2` supported, other models have plans for future support |
| `pg_lb`          |         | `Vector{Real}` | watt  | opf      | Lower bound on active power generation per phase, `size=nphases`                                                                                                                                                |
| `pg_ub`          |         | `Vector{Real}` | watt  | opf      | Upper bound on active power generation per phase, `size=nphases`                                                                                                                                                |
| `qg_lb`          |         | `Vector{Real}` | var   | opf      | Lower bound on reactive power generation per phase, `size=nphases`                                                                                                                                              |
| `qg_ub`          |         | `Vector{Real}` | var   | opf      | Upper bound on reactive power generation per phase, `size=nphases`                                                                                                                                              |
| `pg`             |         | `Vector{Real}` | watt  |          | Present active power generation per phase, `size=nphases`                                                                                                                                                       |
| `qg`             |         | `Vector{Real}` | var   |          | Present reactive power generation per phase, `size=nphases`                                                                                                                                                     |
| `status`         | `1`     | `Bool`         |       | always   | `1` or `0`. Indicates if component is enabled or disabled, respectively                                                                                                                                         |
| `{}_time_series` |         | `Any`          |       |          | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for any parameter                                                                                                     |

#### `generator` Cost Model

The generator cost model is specified by the following fields.

| Name                 | Default           | Type           | Units | Required | Description                                               |
| -------------------- | ----------------- | -------------- | ----- | -------- | --------------------------------------------------------- |
| `cost_pg_model`      | `2`               | `Int`          |       | opf      | Cost model type, `1` = piecewise-linear, `2` = polynomial |
| `cost_pg_parameters` | `[0.0, 1.0, 0.0]` | `Vector{Real}` | $/MVA | opf      | Cost model polynomial                                     |

### Photovoltaic Systems (`solar`)

| Name             | Default | Type           | Units | Required | Description                                                                                                 |
| ---------------- | ------- | -------------- | ----- | -------- | ----------------------------------------------------------------------------------------------------------- |
| `bus`            |         | `Any`          |       | always   | id of bus connection                                                                                        |
| `connections`    |         | `Vector{Any}`  |       | always   | Ordered list of connected conductors                                                                        |
| `configuration`  | `"wye"` | `String`       |       | always   | `"wye"` or `"delta"`. If `"wye"`, `connections[end]=neutral`                                                |
| `pg_lb`          |         | `Vector{Real}` | watt  | opf      | Lower bound on active power generation per phase, `size=nphases`                                            |
| `pg_ub`          |         | `Vector{Real}` | watt  | opf      | Upper bound on active power generation per phase, `size=nphases`                                            |
| `qg_lb`          |         | `Vector{Real}` | var   | opf      | Lower bound on reactive power generation per phase, `size=nphases`                                          |
| `qg_ub`          |         | `Vector{Real}` | var   | opf      | Upper bound on reactive power generation per phase, `size=nphases`                                          |
| `pg`             |         | `Vector{Real}` | watt  |          | Present active power generation per phase, `size=nphases`                                                   |
| `qg`             |         | `Vector{Real}` | var   |          | Present reactive power generation per phase, `size=nphases`                                                 |
| `status`         | `1`     | `Bool`         |       | always   | `1` or `0`. Indicates if component is enabled or disabled, respectively                                     |
| `{}_time_series` |         | `Any`          |       |          | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for any parameter |

#### Irradiation Model

#### `solar` Cost Model

The cost model for a photovoltaic system currently matches that of generators.

| Name                 | Default           | Type           | Units | Required | Description                                               |
| -------------------- | ----------------- | -------------- | ----- | -------- | --------------------------------------------------------- |
| `cost_pg_model`      | `2`               | `Int`          |       | opf      | Cost model type, `1` = piecewise-linear, `2` = polynomial |
| `cost_pg_parameters` | `[0.0, 1.0, 0.0]` | `Vector{Real}` | $/MVA | opf      | Cost model polynomial                                     |

### Wind Turbine Systems (`wind`)

Wind turbine systems are most closely approximated by induction machines, also known as asynchornous machines. These are not currently supported, but there is plans to support them in the future.

### Storage (`storage`)

A storage object is a flexible component that can represent a variety of energy storage objects, like Li-ion batteries, hydrogen fuel cells, flywheels, etc.

- How to include the inverter model for this? Similar issue as for a PV generator

| Name                   | Default | Type           | Units   | Required | Description                                                                                                 |
| ---------------------- | ------- | -------------- | ------- | -------- | ----------------------------------------------------------------------------------------------------------- |
| `bus`                  |         | `Any`          |         | always   | id of bus connection                                                                                        |
| `connections`          |         | `Vector{Any}`  |         | always   | Ordered list of connected conductors                                                                        |
| `configuration`        | `"wye"` | `String`       |         | always   | `"wye"` or `"delta"`. If `"wye"`, `connections[end]=neutral`                                                |
| `energy`               |         | `Real`         | watt-hr | always   | Stored energy                                                                                               |
| `energy_ub`            |         | `Real`         |         | opf      |                                                                                                             |
| `charge_ub`            |         | `Real`         |         | opf      |                                                                                                             |
| `discharge_ub`         |         | `Real`         |         | opf      |                                                                                                             |
| `sm_ub`                |         | `Vector{Real}` | watt    | opf      | Power rating, `size=nphases`                                                                                |
| `cm_ub`                |         | `Vector{Real}` | amp     | opf      | Current rating, `size=nphases`                                                                              |
| `charge_efficiency`    |         | `Real`         |         | always   |                                                                                                             |
| `discharge_efficiency` |         | `Real`         |         | always   |                                                                                                             |
| `qs_ub`                |         | `Vector{Real}` |         | opf      | Maximum reactive power injection, `size=nphases`                                                            |
| `qs_lb`                |         | `Vector{Real}` |         | opf      | Minimum reactive power injection, `size=nphases`                                                            |
| `rs`                   |         | `Vector{Real}` | ohm     | always   |                                                                                                             |
| `xs`                   |         | `Vector{Real}` | ohm     | always   |                                                                                                             |
| `pex`                  |         | `Real`         |         | always   | Total active power standby exogenous flow (loss)                                                            |
| `qex`                  |         | `Real`         |         | always   | Total reactive power standby exogenous flow (loss)                                                          |
| `ps`                   |         | `Vector{Real}` | watt    |          | Present active power injection                                                                              |
| `qs`                   |         | `Vector{Real}` | var     |          | Present reactive power injection                                                                            |
| `status`               | `1`     | `Bool`         |         | opf      | `1` or `0`. Indicates if component is enabled or disabled, respectively                                     |
| `{}_time_series`       |         | `Any`          |         |          | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for any parameter |

### Voltage Sources (`voltage_source`)

A voltage source is a source of power at a set voltage magnitude and angle connected to a slack bus. If `rs` or `xs` are not specified, the voltage source is assumed to be lossless, otherwise virtual `branch` and `bus` will be created in the mathematical model to represent the internal losses of the voltage source.

| Name             | Default         | Type           | Units  | Required | Description                                                                                                 |
| ---------------- | --------------- | -------------- | ------ | -------- | ----------------------------------------------------------------------------------------------------------- |
| `bus`            |                 | `Any`          |        | always   | id of bus connection                                                                                        |
| `connections`    |                 | `Vector{Any}`  |        | always   | Ordered list of connected conductors                                                                        |
| `configuration`  | `"wye"`         | `String`       |        | always   | `"wye"` or `"delta"`. If `"wye"`, `connections[end]=neutral`                                                |
| `vm`             | `ones(nphases)` | `Vector{Real}` | volt   | always   | Voltage magnitude set at slack bus                                                                          |
| `va`             | `0.0`           | `Real`         | degree |          | Voltage angle offset at slack bus                                                                           |
| `rs`             |                 | `Matrix{Real}` | ohm    |          | Internal series resistance of voltage source, `size=(nconductors,nconductors)`                              |
| `xs`             |                 | `Matrix{Real}` | ohm    |          | Internal series reactance of voltage soure, `size=(nconductors,nconductors)`                                |
| `status`         | `1`             | `Bool`         |        | always   | `1` or `0`. Indicates if component is enabled or disabled, respectively                                     |
| `{}_time_series` |                 | `Any`          |        |          | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for any parameter |

## Data Objects (codes, curves, shapes)

These objects are referenced by node and edge objects, but are not part of the network themselves, only containing data.

### Linecodes (`linecode`)

Linecodes are easy ways to specify properties common to multiple lines.

- Should the linecode also include a `cm_ub` and `sm_ub`? I think not. `cm_ub` yes, `sm_ub` does not make sense. `sm_ub`, if desired, should be specified at the line level
- Does `cmatrix` makes more sense than `g_fr, b_fr, g_to, b_to` when parametrized per length? No, enforce symmetry. If needed place a shunt or inject at low-level.

| Name             | Default | Type           | Units            | Required | Description                                                                                                 |
| ---------------- | ------- | -------------- | ---------------- | -------- | ----------------------------------------------------------------------------------------------------------- |
| `rs`             |         | `Matrix{Real}` | ohm/meter        |          | Series resistance, `size=(nconductors,nconductors)`                                                         |
| `xs`             |         | `Matrix{Real}` | ohm/meter        |          | Series reactance, `size=(nconductors,nconductors)`                                                          |
| `g_fr`           |         | `Matrix{Real}` | siemens/meter/Hz |          | From-side conductance, `size=(nconductors,nconductors)`                                                     |
| `b_fr`           |         | `Matrix{Real}` | siemens/meter/Hz |          | From-side susceptance, `size=(nconductors,nconductors)`                                                     |
| `g_to`           |         | `Matrix{Real}` | siemens/meter/Hz |          | To-side conductance, `size=(nconductors,nconductors)`                                                       |
| `b_to`           |         | `Matrix{Real}` | siemens/meter/Hz |          | To-side susceptance, `size=(nconductors,nconductors)`                                                       |
| `cm_ub`          |         | `Vector{Real}` | ampere           |          | maximum current per conductor, symmetrically applicable                                                     |
| `{}_time_series` |         | `Any`          |                  |          | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for any parameter |

### Transformer Codes (`xfmrcode`)

Transformer codes are easy ways to specify properties common to multiple transformers

| Name             | Default                                | Type                   | Units | Required | Description                                                                                                    |
| ---------------- | -------------------------------------- | ---------------------- | ----- | -------- | -------------------------------------------------------------------------------------------------------------- |
| `configurations` | `fill("wye", nwindings)`               | `Vector{String}`       |       |          | `"wye"` or `"delta"`. List of configuration for each winding, `size=nwindings`                                 |
| `xsc`            | `[0.0]`                                | `Vector{Real}`         | ohm   |          | List of short-circuit reactances between each pair of windings; enter as a list of the upper-triangle elements |
| `rs`             | `zeros(nwindings)`                     | `Vector{Real}`         | ohm   |          | List of the winding resistance for each winding                                                                |
| `tm_nom`         | `ones(nwindings)`                      | `Vector{Real}`         |       |          | Nominal tap ratio for the transformer, `size=nwindings` (multiplier)                                           |
| `tm_ub`          |                                        | `Vector{Vector{Real}}` |       |          | Maximum tap ratio for each winding and phase, `size=((nphases), nwindings)` (base=`tm_nom`)                    |
| `tm_lb`          |                                        | `Vector{Vector{Real}}` |       |          | Minimum tap ratio for for each winding and phase, `size=((nphases), nwindings)` (base=`tm_nom`)                |
| `tm_set`         | `fill(fill(1.0, nphases), nwindings)`  | `Vector{Vector{Real}}` |       |          | Set tap ratio for each winding and phase, `size=((nphases), nwindings)` (base=`tm_nom`)                        |
| `tm_fix`         | `fill(fill(true, nphases), nwindings)` | `Vector{Vector{Bool}}` |       |          | Indicates for each winding and phase whether the tap ratio is fixed, `size=((nphases), nwindings)`             |
| `{}_time_series` |                                        | `Any`                  |       |          | id of `time_series` object that will replace the values of parameter given by `{}`. Valid for any parameter    |

### Voltage Zones (`voltage_zone`)

Voltage zones are a convenient way to specify voltage bounds for a set of three-phase buses (ones with `phases`/`neutral` properties).

| Name       | Default | Type   | Units | Required | Description                                               |
| ---------- | ------- | ------ | ----- | -------- | --------------------------------------------------------- |
| `vnom`     |         | `Real` |       | always   | Nominal phase-to-neutral voltage                          |
| `vm_pn_lb` |         | `Real` |       |          | Minimum phase-to-neutral voltage magnitude for all phases |
| `vm_pn_ub` |         | `Real` |       |          | Maximum phase-to-neutral voltage magnitude for all phases |
| `vm_pp_lb` |         | `Real` |       |          | Minimum phase-to-phase voltage magnitude for all phases   |
| `vm_pp_ub` |         | `Real` |       |          | Maximum phase-to-phase voltage magnitude for all phases   |
| `vm_ng_ub` |         | `Real` |       |          | Maximum neutral-to-ground voltage magnitude               |

### Curves (`curve`)

Curve objects are functions f(x) that return single values for a given x. This is useful for several objects like `solar` power-temperature curves, or efficiency curves on various objects.

| Name    | Default | Type     | Units | Required | Description |
| ------- | ------- | -------- | ----- | -------- | ----------- |
| `curve` |         | Function |       | always   |             |

### Time Series (`time_series`)

Time series objects are used to specify time series for _e.g._ load or generation forecasts.

Every parameter for any component specified in this document can support a time series by appending `_time_series` to the parameter name. For example, for a `load`, if `pd_nom` is supposed to be a time series, the user would specify `"pd_nom_time_series" => time_series_id` where `time_series_id` is the `id` of an object in `time_series`, and has type `Any`.

| Name         | Default | Type           | Units | Required | Description                                                                                                                                                         |
| ------------ | ------- | -------------- | ----- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `time`       |         | `Vector{Real}` | hour  | always   | Time points at which values are specified                                                                                                                           |
| `multiplier` |         | `Vector{Real}` |       | always   | Multipers at each time step given in `time`. If `useactual`, multipliers do not multiply original values, but replace them on the objects to which they are applied |
