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
            "field" => value,
            ...
        ),
        ...
    ),
    ...
)
```

Valid component types are those that are documented in the sectios below. Each component object is identified by an `id`, which can be any immutable value, but `id` does not appear inside of the component dictionary, and only appears as keys to the component dictionaries under each component type.

Each edge or node component (_i.e._ all those that are not data objects or buses), are expected to have `status` fields to specify whether the component is active or disabled, `bus` or `f_bus` and `t_bus`, to specify the buses that are connected to the component, and `connections` or `f_connections` and `t_connections`, to specify the terminals of the buses that are actively connected in an ordered list. Terminals/connections can be any immutable value, as can bus ids.

Parameter values on components are expected to be specified in SI units by default (where applicable) in the engineering data model. Relevant expected units are noted in the sections below.

TODO: Specify units on each component

## Root-Level Properties

At the root level of the data structure, the following fields can be found.

| Name         | Default         | Type                 | Description                                                                                                                |
| ------------ | --------------- | -------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `name`       |                 | `String`             | (Optional) Case name                                                                                                       |
| `data_model` | `"engineering"` | `String`             | `"engineering"`, `"mathematical"`, or `"dss"`. Type of the data model (this document describes `data_model="engineering"`) |
| `settings`   | `Dict()`        | `Dict{String,<:Any}` | Base settings for the data model, see Settings section below for details                                                   |

## Settings (`settings`)

At the root-level of the data model a `settings` dictionary object is expected, containing the following fields.

| Name             | Default | Type   | Units | Description                                                    |
| ---------------- | ------- | ------ | ----- | -------------------------------------------------------------- |
| `v_var_scalar`   | `1e3`   | `Real` |       | Scalar for voltage values                                      |
| `vbase`          |         | `Real` |       | Voltage base (_i.e._ basekv) at `base_bus`                     |
| `sbase`          |         | `Real` |       | Power base (baseMVA) at `base_bus`                             |
| `base_bus`       |         | `Any`  |       | id of bus at which `vbase` and `sbase` apply                   |
| `base_frequency` | `60.0`  | `Real` | Hz    | Frequency base, _i.e._ the base frequency of the whole circuit |

## Buses (`bus`)

The data model below allows us to include buses of arbitrary many terminals (_i.e._, more than the usual four). This would be useful for

- underground lines with multiple neutrals which are not joined at every bus;
- distribution lines that carry several conventional lines in parallel (see for example the quad circuits in NEVTestCase).

| Name          | Default     | Type            | Units  | Description                                                                                          |
| ------------- | ----------- | --------------- | ------ | ---------------------------------------------------------------------------------------------------- |
| `terminals`   | `[1,2,3,4]` | `Vector{Any}`   |        | Terminals for which the bus has active connections                                                   |
| `neutral`     | `4`         | `Any`           |        | Identifies the terminal that represents the neutral conductor (only one neutral conductor supported) |
| `vm_min`      |             | `Vector{Real}`  |        | Minimum conductor-to-ground voltage magnitude, `size=nphases`                                        |
| `vm_max`      |             | `Vector{Real}`  |        | Maximum conductor-to-ground voltage magnitude, `size=nphases`                                        |
| `vm_pair_max` |             | `Vector{Tuple}` |        | _e.g._  `[(1,2,210)]` means \|U1-U2\|>210                                                            |
| `vm_pair_min` |             | `Vector{Tuple}` |        | _e.g._  `[(1,2,230)]` means \|U1-U2\|<230                                                            |
| `grounded`    | `[]`        | `Vector{Any}`   |        | List of terminals which are grounded                                                                 |
| `rg`          | `[]`        | `Vector{Real}`  |        | Resistance of each defined grounding, `size=length(grounded)`                                        |
| `xg`          | `[]`        | `Vector{Real}`  |        | Reactance of each defined grounding, `size=length(grounded)`                                         |
| `vm`          |             | `Vector{Real}`  | volt   | (Optional) Voltage magnitude at bus                                                                  |
| `va`          |             | `Vector{Real}`  | degree | (Optional) Voltage angle at bus                                                                      |
| `status`      | `1`         | `Bool`          |        | `1` or `0`. Indicates if component is enabled or disabled, respectively                              |

The tricky part is how to encode bounds for these type of buses. The most general is defining a list of three-tuples. Take for example a typical bus in a three-phase, four-wire network, where `terminals=[a,b,c,n]`. Such a bus might have

- phase-to-neutral bounds `vm_pn_max=250`, `vm_pn_min=210`
- and phase-to-phase bounds `vm_pp_max=440`, `vm_pp_min=360`.

We can then define this equivalently as

- `vm_pair_max = [(a,n,250), (b,n,250), (c,n,250), (a,b,440), (b,c,440), (c,a,440)]`
- `vm_pair_min = [(a,n,210), (b,n,210), (c,n,210), (a,b,360), (b,c,360), (c,a,360)]`

If terminal `4` is grounding through an impedance `Z=1+j2`, we write

- `grounded=[4]`, `rg=[1]`, `xg=[2]`

Since this might be confusing for novice users, we also allow the user to define bounds through the following component.

### Special Case: three-phase bus

- Specify bounds relative to some `vnom`? Yes, but in voltage_zones

Much of the difficulty arrises from supporting phase-to-phase bounds for both two-phase and three-phase systems. For example, a 2-phase `[a,b]` system only has 1 phase-to-phase bound, whilst a 3-phase system `[a,b,c]` has 3!

A 4-phase system `[a,b,c,d]` is not well-defined; should there be phase-to-phase bounds for each permutation, _i.e._, `[(a,b), (a,c), (a,d), (b,c), (b,d), (c,d)]`, or only for adjacent ones as it would apply to a 4-phase delta connection, `[(a,b), (b,c), (c,d), (a,d)]`? For three-phase systems these two options coincide.

In order to avoid all of this confusion, we introduce this component, which is at most 3-phase, `|phases|<=3`, and which specifies all bounds symmetrically. For example,

- `phases=[1,2,3], vm_pp_max=440` implies |U1-U2|, |U2-U3|, |U1-U3| <= 440
- `phases=[1,2], vm_pn_max=440` implies |U1-U2| <= 440

This keeps the user from specifying things that do not make sense. This type of bus would suffice for most of the use cases.

| Name           | Default | Type   | Units | Description                                               |
| -------------- | ------- | ------ | ----- | --------------------------------------------------------- |
| `vm_pn_min`    |         | `Real` |       | Minimum phase-to-neutral voltage magnitude for all phases |
| `vm_pn_max`    |         | `Real` |       | Maximum phase-to-neutral voltage magnitude for all phases |
| `vm_pp_min`    |         | `Real` |       | Minimum phase-to-phase voltage magnitude for all phases   |
| `vm_pp_max`    |         | `Real` |       | Maximum phase-to-phase voltage magnitude for all phases   |
| `vm_ng_rating` |         | `Real` |       | Maximum neutral-to-ground voltage magnitude               |

Should we automatically imply vmax and vmin bounds for this?, _i.e._

- `vmax = [fill(vm_pn_max+vm_ng_rating, |phases|)..., vm_ng_rating]`
- `vmin = [fill(vm_pn_min-vm_ng_rating, |phases|)..., 0]`

## Edge Objects

These objects represent edges on the power grid and therefore require `f_bus` and `t_bus` (or `buses` in the case of transformers), and `f_connections` and `t_connections` (or `connections` in the case of transformers).

### Lines (`line`)

This is a pi-model branch. When a `linecode` is given, and any of `rs`, `xs`, `b_fr`, `b_to`, `g_fr` or `g_to` are specified, any of those overwrite the values on the linecode.

| Name            | Default | Type           | Units            | Description                                                                |
| --------------- | ------- | -------------- | ---------------- | -------------------------------------------------------------------------- |
| `f_bus`         |         | `Any`          |                  | id of from-side bus connection                                             |
| `t_bus`         |         | `Any`          |                  | id of to-side bus connection                                               |
| `f_connections` |         | `Vector{Any}`  |                  | Indicates for each conductor, to which terminal of the `f_bus` it connects |
| `t_connections` |         | `Vector{Any}`  |                  | Indicates for each conductor, to which terminal of the `t_bus` it connects |
| `linecode`      |         | `Any`          |                  | (Optional) id of an associated linecode                                    |
| `rs`            |         | `Matrix{Real}` | ohm/meter        | Series resistance matrix, `size=(nconductors,nconductors)`                 |
| `xs`            |         | `Matrix{Real}` | ohm/meter        | Series reactance matrix, `size=(nconductors,nconductors)`                  |
| `g_fr`          |         | `Matrix{Real}` | siemens/meter/Hz | From-side conductance, `size=(nconductors,nconductors)`                    |
| `b_fr`          |         | `Matrix{Real}` | siemens/meter/Hz | From-side susceptance, `size=(nconductors,nconductors)`                    |
| `g_to`          |         | `Matrix{Real}` | siemens/meter/Hz | To-side conductance, `size=(nconductors,nconductors)`                      |
| `b_to`          |         | `Matrix{Real}` | siemens/meter/Hz | To-side susceptance, `size=(nconductors,nconductors)`                      |
| `length`        | `1.0`   | `Real`         | meter            | Length of the line                                                         |
| `c_rating`      |         | `Vector{Real}` | amp              | Symmetrically applicable current rating, `size=nconductors`                |
| `s_rating`      |         | `Vector{Real}` | watt             | Symmetrically applicable power rating, `size=nconductors`                  |
| `status`        | `1`     | `Bool`         |                  | `1` or `0`. Indicates if component is enabled or disabled, respectively    |

- which takes presidence, `c_rating` or `s_rating` if both are specified?

Add example for linecodes/lines off different size (conductors)

### Transformers (`transformer`)

These are n-winding (`nwinding`), n-phase (`nphase`), lossy transformers. Note that most properties are now Vectors (or Vectors of Vectors), indexed over the windings.

| Name             | Default                                | Type                   | Units         | Description                                                                                                    |
| ---------------- | -------------------------------------- | ---------------------- | ------------- | -------------------------------------------------------------------------------------------------------------- |
|                  | `buses`                                |                        | `Vector{Any}` |                                                                                                                | List of bus for each winding, `size=nwindings` |
| `connections`    |                                        | `Vector{Vector{Any}}`  |               | List of connection for each winding, `size=((nconductors), nwindings)`                                         |
| `configurations` | `fill("wye", nwindings)`               | `Vector{String}`       |               | `"wye"` or `"delta"`. List of configuration for each winding, `size=nwindings`                                 |
| `xsc`            | `[0.0]`                                | `Vector{Real}`         | ohm           | List of short-circuit reactances between each pair of windings; enter as a list of the upper-triangle elements |
| `rs`             | `zeros(nwindings)`                     | `Vector{Real}`         | ohm           | List of the winding resistance for each winding                                                                |
| `tm_nom`         | `ones(nwindings)`                      | `Vector{Real}`         |               | Nominal tap ratio for the transformer, `size=nwindings` (multiplier)                                           |
| `tm_max`         |                                        | `Vector{Vector{Real}}` |               | Maximum tap ratio for each winding and phase, `size=((nphases), nwindings)` (base=`tm_nom`)                    |
| `tm_min`         |                                        | `Vector{Vector{Real}}` |               | Minimum tap ratio for for each winding and phase, `size=((nphases), nwindings)` (base=`tm_nom`)                |
| `tm_set`         | `fill(fill(1.0, nphases), nwindings)`  | `Vector{Vector{Real}}` |               | Set tap ratio for each winding and phase, `size=((nphases), nwindings)` (base=`tm_nom`)                        |
| `tm_fix`         | `fill(fill(true, nphases), nwindings)` | `Vector{Vector{Bool}}` |               | Indicates for each winding and phase whether the tap ratio is fixed, `size=((nphases), nwindings)`             |
| `status`         | `1`                                    | `Bool`                 |               | `1` or `0`. Indicates if component is enabled or disabled, respectively                                        |

#### Assymetric, Lossless, Two-Winding (AL2W) Transformers

Special case of the Generic transformer. These are transformers are assymetric (A), lossless (L) and two-winding (2W). Assymetric refers to the fact that the secondary is always has a `wye` configuration, whilst the primary can be `delta`. The table below indicates alternate, more simple ways to specify the special case of an AL2W Transformer. `xsc` and `rs` cannot be specified for an AL2W transformer.

| Name            | Default               | Type           | Units | Description                                                                                                        |
| --------------- | --------------------- | -------------- | ----- | ------------------------------------------------------------------------------------------------------------------ |
| `f_bus`         |                       | `Any`          |       | (Optional) Alternative way to specify `buses`, requires both `f_bus` and `t_bus`                                   |
| `t_bus`         |                       | `Any`          |       | (Optional) Alternative  way to specify `buses`, requires both `f_bus` and `t_bus`                                  |
| `f_connections` |                       | `Vector{Any}`  |       | (Optional) Alternative way to specify `connections`, requires both `f_connections` and `t_connections`             |
| `t_connections` |                       | `Vector{Any}`  |       | (Optional) Alternative way to specify `connections`, requires both `f_connections` and `t_connections`             |
| `configuration` | `"wye"`               | `String`       |       | (Optional) `"wye"` or `"delta"`. Alternative way to specify the from-side configuration, to-side is always `"wye"` |
| `tm_nom`        | `1.0`                 | `Real`         |       | Nominal tap ratio for the transformer (multiplier)                                                                 |
| `tm_max`        |                       | `Vector{Real}` |       | Maximum tap ratio for each phase (base=`tm_nom`)                                                                   |
| `tm_min`        |                       | `Vector{Real}` |       | Minimum tap ratio for each phase (base=`tm_nom`)                                                                   |
| `tm_set`        | `fill(1.0, nphases)`  | `Vector{Real}` |       | Set tap ratio for each phase (base=`tm_nom`)                                                                       |
| `tm_fix`        | `fill(true, nphases)` | `Vector{Real}` |       | Indicates for each phase whether the tap ratio is fixed                                                            |

### Switches (`switch`)

Switches without any of `rs`, `xs`, `g_fr`, `b_fr`, `g_to`, `b_to` defined are assumed to be lossless. If lossy parameters are defined, `switch` objects will be decomposed into virtual `branch` & `bus`, and an ideal `switch`.

| Name            | Default    | Type           | Units   | Description                                                                |
| --------------- | ---------- | -------------- | ------- | -------------------------------------------------------------------------- |
| `f_bus`         |            | `Any`          |         | id of from-side bus connection                                             |
| `t_bus`         |            | `Any`          |         | id of to-side bus connection                                               |
| `f_connections` |            | `Vector{Any}`  |         | Indicates for each conductor, to which terminal of the `f_bus` it connects |
| `t_connections` |            | `Vector{Any}`  |         | Indicates for each conductor, to which terminal of the `t_bus` it connects |
| `c_rating`      |            | `Vector{Real}` | amp     | Symmetrically applicable current rating                                    |
| `s_rating`      |            | `Vector{Real}` | watt    | Symmetrically applicable power rating                                      |
| `rs`            |            | `Matrix{Real}` | ohm     | (Optional) Series resistance matrix, `size=(nphases,nphases)`              |
| `xs`            |            | `Matrix{Real}` | ohm     | (Optional) Series reactance matrix, `size=(nphases,nphases)`               |
| `g_fr`          |            | `Matrix{Real}` | siemens | (Optional) From-side conductance, `size=(nphases,nphases)`,                |
| `b_fr`          |            | `Matrix{Real}` | siemens | (Optional) From-side susceptance, `size=(nphases,nphases)`                 |
| `g_to`          |            | `Matrix{Real}` | siemens | (Optional) To-side conductance, `size=(nphases,nphases)`                   |
| `b_to`          |            | `Matrix{Real}` | siemens | (Optional) To-side susceptance, `size=(nphases,nphases)`                   |
| `state`         | `"closed"` | `String`       |         | `"open"` or `"closed"`, to indicate initial/current state of switch        |
| `status`        | `1`        | `Bool`         |         | `1` or `0`. Indicates if component is enabled or disabled, respectively    |

#### Fuses (`fuse`)

Special Case of switch, with its own separate component category for easier tracking. Shares the fields of switch, with these additions.

| Name                    | Default | Type     | Units | Description                                                                        |
| ----------------------- | ------- | -------- | ----- | ---------------------------------------------------------------------------------- |
| `fuse_curve`            |         | `String` |       | (Optional) id of `curve` object that specifies the fuse blowing condition          |
| `minimum_melting_curve` |         | `String` |       | (Optional) id of `curve` that specifies the minimum melting conditions of the fuse |

### Line Reactors (`line_reactor`)

TODO

### Series Capacitors (`series_capacitor`)

TODO

## Node Objects

These are objects that have single bus connections. Every object will have at least `bus`, `connections`, and `status`.

### Shunts (`shunt`)

| Name            | Default | Type           | Units   | Description                                                             |
| --------------- | ------- | -------------- | ------- | ----------------------------------------------------------------------- |
| `bus`           |         | `Any`          |         | id of bus connection                                                    |
| `connections`   |         | `Vector{Any}`  |         | Ordered list of connected conductors                                    |
| `configuration` | `"wye"` | `String`       |         | `"wye"` or `"delta"`. If `"wye"`, `connections[end]=neutral`            |
| `g`             |         | `Matrix{Real}` | siemens | Conductance, `size=(nphases,nphases)`                                   |
| `b`             |         | `Matrix{Real}` | siemens | Susceptance, `size=(nphases,nphases)`                                   |
| `status`        | `1`     | `Bool`         |         | `1` or `0`. Indicates if component is enabled or disabled, respectively |

#### Shunt Capacitors (`shunt_capacitor`)

This is a special case of `shunt` with its own data category for easier tracking.

| Name            | Default | Type           | Units   | Description                                                                                          |
| --------------- | ------- | -------------- | ------- | ---------------------------------------------------------------------------------------------------- |
| `bus`           |         | `Any`          |         | id of bus connection                                                                                 |
| `connections`   |         | `Vector{Any}`  |         | Ordered list of connected conductors, `size=nconductors`                                             |
| `configuration` | `"wye"` | `String`       |         | `"wye"` or `"delta"`. If `"wye"`, `connections[end]=neutral`. For `"delta"`, 2 or 3 connections only |
| `b`             |         | `Vector{Real}` | siemens | Conductance, `size=(nconductors,nconductors)`                                                        |
| `vnom`          |         | `Real`         | volt    | Nominal voltage (multiplier)                                                                         |

Give some examples

#### Shunt Reactors (`shunt_reactor`)

This is a special case of `shunt` with its own data category for easier tracking.

### Loads (`load`)

| Name            | Default            | Type           | Units | Description                                                                                                                             |
| --------------- | ------------------ | -------------- | ----- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `bus`           |                    | `Any`          |       | id of bus connection                                                                                                                    |
| `connections`   |                    | `Vector{Any}`  |       | Ordered list of connected conductors                                                                                                    |
| `configuration` | `"wye"`            | `String`       |       | `"wye"` or "`delta`". If `"wye"`, `connections[end]=neutral`                                                                            |
| `model`         | `"constant_power"` | `String`       |       | `"constant_power"`, `"constant_impedance"`, `"constant_current"`, `"exponential"`, or `"zip"`. Indicates the type of voltage-dependency |
| `pd_ref`        |                    | `Vector{Real}` | watt  | Real load, `size=nphases`                                                                                                               |
| `qd_ref`        |                    | `Vector{Real}` | var   | Reactive load, `size=nphases`                                                                                                           |
| `vnom`          |                    | `Real`         | volt  | Nominal voltage (multiplier)                                                                                                            |
| `status`        | `1`                | `Bool`         |       | `1` or `0`. Indicates if component is enabled or disabled, respectively                                                                 |

#### `model="exponential"`

| Name    | Default | Type   | Units | Description |
| ------- | ------- | ------ | ----- | ----------- |
| `exp_p` |         | `Real` |       |             |
| `exp_q` |         | `Real` |       |             |

#### `model="zip"`

| Name        | Default | Type   | Units | Description |
| ----------- | ------- | ------ | ----- | ----------- |
| `coeff_z_p` |         | `Real` |       |             |
| `coeff_i_p` |         | `Real` |       |             |
| `coeff_p_p` |         | `Real` |       |             |
| `coeff_z_q` |         | `Real` |       |             |
| `coeff_i_q` |         | `Real` |       |             |
| `coeff_p_q` |         | `Real` |       |             |

### Generators or Induction Machines or Asychronous Machines? (`generator` or `asychronous_machine` or `induction_machine`?)

| Name            | Default | Type           | Units | Description                                                             |
| --------------- | ------- | -------------- | ----- | ----------------------------------------------------------------------- |
| `bus`           |         | `Any`          |       | id of bus connection                                                    |
| `connections`   |         | `Vector{Any}`  |       | Ordered list of connected conductors                                    |
| `configuration` | `"wye"` | `String`       |       | `"wye"` or `"delta"`. If `"wye"`, `connections[end]=neutral`            |
| `pg_min`        |         | `Vector{Real}` | watt  | Lower bound on active power generation per phase, `size=nphases`        |
| `pg_max`        |         | `Vector{Real}` | watt  | Upper bound on active power generation per phase, `size=nphases`        |
| `qg_min`        |         | `Vector{Real}` | var   | Lower bound on reactive power generation per phase, `size=nphases`      |
| `qg_max`        |         | `Vector{Real}` | var   | Upper bound on reactive power generation per phase, `size=nphases`      |
| `status`        | `1`     | `Bool`         |       | `1` or `0`. Indicates if component is enabled or disabled, respectively |

#### Cost Model

The generator cost model is specified by the following fields.

| Name              | Default           | Type           | Units | Description           |
| ----------------- | ----------------- | -------------- | ----- | --------------------- |
| `cost_model`      | `2`               | `Int`          |       |                       |
| `startup`         | `0.0`             | `Real`         | $     | Startup cost          |
| `shutdown`        | `0.0`             | `Real`         | $     | Shutdown cost         |
| `ncost_terms`     | `3`               | `Int`          |       | Number of cost terms  |
| `cost_polynomial` | `[0.0, 1.0, 0.0]` | `Vector{Real}` | $/MVA | Cost model polynomial |

### Photovoltaic Systems (`solar`)

TODO

### Wind Turbine Systems (`wind`)

TODO

### Storage (`storage`)

A storage object is a flexible component that can represent a variety of energy storage objects, like Li-ion batteries, hydrogen fuel cells, flywheels, etc.

- How to include the inverter model for this? Similar issue as for a PV generator

| Name                   | Default | Type           | Units   | Description                                                             |
| ---------------------- | ------- | -------------- | ------- | ----------------------------------------------------------------------- |
| `bus`                  |         | `Any`          |         | id of bus connection                                                    |
| `connections`          |         | `Vector{Any}`  |         | Ordered list of connected conductors                                    |
| `configuration`        | `"wye"` | `String`       |         | `"wye"` or `"delta"`. If `"wye"`, `connections[end]=neutral`            |
| `energy`               |         | `Real`         | watt-hr | Stored energy                                                           |
| `energy_rating`        |         | `Real`         |         |                                                                         |
| `charge_rating`        |         | `Real`         |         |                                                                         |
| `discharge_rating`     |         | `Real`         |         |                                                                         |
| `s_rating`             |         | `Vector{Real}` | watt    | Power rating, `size=nphases`                                            |
| `c_rating`             |         | `Vector{Real}` | amp     | Current rating, `size=nphases`                                          |
| `charge_efficiency`    |         | `Real`         |         |                                                                         |
| `discharge_efficiency` |         | `Real`         |         |                                                                         |
| `qmax`                 |         | `Vector{Real}` |         | Maximum reactive power, `size=nphases`                                  |
| `qmin`                 |         | `Vector{Real}` |         | Minimum reactive power, `size=nphases`                                  |
| `r`                    |         | `Vector{Real}` | ohm     |                                                                         |
| `x`                    |         | `Vector{Real}` | ohm     |                                                                         |
| `p_loss`               |         | `Real`         |         | Total active power standby loss                                         |
| `q_loss`               |         | `Real`         |         | Total reactive power standby loss                                       |
| `status`               | `1`     | `Bool`         |         | `1` or `0`. Indicates if component is enabled or disabled, respectively |

### Voltage Sources (`voltage_source`)

A voltage source is a source of power at a set voltage magnitude and angle connected to a slack bus. If `rs` or `xs` are not specified, the voltage source is assumed to be lossless, otherwise virtual `branch` and `bus` will be created in the mathematical model to represent the internal losses of the voltage source.

| Name            | Default         | Type           | Units  | Description                                                                               |
| --------------- | --------------- | -------------- | ------ | ----------------------------------------------------------------------------------------- |
| `bus`           |                 | `Any`          |        | id of bus connection                                                                      |
| `connections`   |                 | `Vector{Any}`  |        | Ordered list of connected conductors                                                      |
| `configuration` | `"wye"`         | `String`       |        | `"wye"` or `"delta"`. If `"wye"`, `connections[end]=neutral`                              |
| `vm`            | `ones(nphases)` | `Vector{Real}` | volt   | Voltage magnitude set at slack bus                                                        |
| `va`            | `0.0`           | `Real`         | degree | (Optional) Voltage angle offset at slack bus                                              |
| `rs`            |                 | `Matrix{Real}` | ohm    | (Optional) Internal series resistance of voltage source, `size=(nconductors,nconductors)` |
| `xs`            |                 | `Matrix{Real}` | ohm    | (Optional) Internal series reactance of voltage soure, `size=(nconductors,nconductors)`   |
| `status`        | `1`             | `Bool`         |        | `1` or `0`. Indicates if component is enabled or disabled, respectively                   |

## Data Objects (codes, curves, shapes)

These objects are referenced by node and edge objects, but are not part of the network themselves, only containing data.

### Linecodes (`linecode`)

Linecodes are easy ways to specify properties common to multiple lines.

- Should the linecode also include a `c_rating` and `s_rating`? I think not. `c_rating` yes, `s_rating` does not make sense. `s_rating`, if desired, should be specified at the line level
- Does `cmatrix` makes more sense than `g_fr, b_fr, g_to, b_to` when parametrized per length? No, enforce symmetry. If needed place a shunt or inject at low-level.

| Name     | Default | Type           | Units            | Description                                                             |
| -------- | ------- | -------------- | ---------------- | ----------------------------------------------------------------------- |
| `rs`     |         | `Matrix{Real}` | ohm/meter        | Series resistance, `size=(nconductors,nconductors)`                     |
| `xs`     |         | `Matrix{Real}` | ohm/meter        | Series reactance, `size=(nconductors,nconductors)`                      |
| `g_fr`   |         | `Matrix{Real}` | siemens/meter/Hz | From-side conductance, `size=(nconductors,nconductors)`                 |
| `b_fr`   |         | `Matrix{Real}` | siemens/meter/Hz | From-side susceptance, `size=(nconductors,nconductors)`                 |
| `g_to`   |         | `Matrix{Real}` | siemens/meter/Hz | To-side conductance, `size=(nconductors,nconductors)`                   |
| `b_to`   |         | `Matrix{Real}` | siemens/meter/Hz | To-side susceptance, `size=(nconductors,nconductors)`                   |
| `status` | `1`     | `Bool`         |                  | `1` or `0`. Indicates if component is enabled or disabled, respectively |

### Transformer Codes (`xfmrcode`)

Transformer codes are easy ways to specify properties common to multiple transformers

| Name             | Default                                | Type                   | Units | Description                                                                                                    |
| ---------------- | -------------------------------------- | ---------------------- | ----- | -------------------------------------------------------------------------------------------------------------- |
| `configurations` | `fill("wye", nwindings)`               | `Vector{String}`       |       | `"wye"` or `"delta"`. List of configuration for each winding, `size=nwindings`                                 |
| `xsc`            | `[0.0]`                                | `Vector{Real}`         | ohm   | List of short-circuit reactances between each pair of windings; enter as a list of the upper-triangle elements |
| `rs`             | `zeros(nwindings)`                     | `Vector{Real}`         | ohm   | List of the winding resistance for each winding                                                                |
| `tm_nom`         | `ones(nwindings)`                      | `Vector{Real}`         |       | Nominal tap ratio for the transformer, `size=nwindings` (multiplier)                                           |
| `tm_max`         |                                        | `Vector{Vector{Real}}` |       | Maximum tap ratio for each winding and phase, `size=((nphases), nwindings)` (base=`tm_nom`)                    |
| `tm_min`         |                                        | `Vector{Vector{Real}}` |       | Minimum tap ratio for for each winding and phase, `size=((nphases), nwindings)` (base=`tm_nom`)                |
| `tm_set`         | `fill(fill(1.0, nphases), nwindings)`  | `Vector{Vector{Real}}` |       | Set tap ratio for each winding and phase, `size=((nphases), nwindings)` (base=`tm_nom`)                        |
| `tm_fix`         | `fill(fill(true, nphases), nwindings)` | `Vector{Vector{Bool}}` |       | Indicates for each winding and phase whether the tap ratio is fixed, `size=((nphases), nwindings)`             |
| `status`         | `1`                                    | `Bool`                 |       | `1` or `0`. Indicates if component is enabled or disabled, respectively                                        |

### Curves (`curve`)

Curve objects are functions f(x) that return single values for a given x. This is useful for several objects like `solar` power-temperature curves, or efficiency curves on various objects.

| Name    | Default | Type     | Units | Description |
| ------- | ------- | -------- | ----- | ----------- |
| `curve` |         | Function |       |             |

### Time Series (`time_series`)

Time series objects are used to specify time series for _e.g._ load or generation forecasts.

| Name         | Default | Type           | Units | Description                                                                                                                                                         |
| ------------ | ------- | -------------- | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `time`       |         | `Vector{Real}` | hour  | Time points at which values are specified                                                                                                                           |
| `multiplier` |         | `Vector{Real}` |       | Multipers at each time step given in `time`. If `useactual`, multipliers do not multiply original values, but replace them on the objects to which they are applied |
| `use_actual` | `false` | `Bool`         |       | If `use_actual`, multiplier replaces values, instead of multiplying                                                                                                 |
