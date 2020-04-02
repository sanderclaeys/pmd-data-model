# Data model

## Generic bus

The data model below allows us to include buses of arbitrary many terminals (i.e., more than the usual four). This would be useful for

- underground lines with multiple neutrals which are not joined at every bus;
- distribution lines that carry several conventional lines in parallel (see for example the quad circuits in NEVTestCase).

| name      | default   | type          | description                                   |
| --------- | --------- | ------------- | --------------------------------------------- |
| terminals | [1,2,3,4] | Vector        |                                               |
| vm_max    | /         | Vector{Real}  | maximum conductor-to-ground voltage magnitude |
| vm_min    | /         | Vector{Real}  | minimum conductor-to-ground voltage magnitude |
| vm_pair_max | /         | Vector{Tuple} | e.g.  [(1,2,210)] means \|U1-U2\|>210         |
| vm_pair_min | /         | Vector{Tuple} | e.g.  [(1,2,230)] means \|U1-U2\|<230         |
| grounded  | []        | Vector         | a list of terminals which are grounded        |
| rg        | []        | Vector{Real}  | resistance of each defined grounding          |
| xg        | []        | Vector{Real}  | reactance of each defined grounding           |

The tricky part is how to encode bounds for these type of buses. The most general is defining a list of three-tuples. Take for example a typical bus in a three-phase, four-wire network, where `terminals=[a,b,c,n]`. Such a bus might have

- phase-to-neutral bounds `vm_pn_max=250`, `vm_pn_min=210`
- and phase-to-phase bounds `vm_pp_max=440`, `vm_pp_min=360`.

We can then define this equivalently as

- `vm_pair_max = [(a,n,250), (b,n,250), (c,n,250), (a,b,440), (b,c,440), (c,a,440)]`
- `vm_pair_min = [(a,n,210), (b,n,210), (c,n,210), (a,b,360), (b,c,360), (c,a,360)]`

If terminal `4` is grounding through an impedance `Z=1+j2`, we write
- `grounded=[4]`, `rg=[1]`, `xg=[2]`

Since this might be confusing for novice users, we also allow the user to define bounds through the following component.

## Three-phase bus

- Think of a better name!
- Specify bounds relative to some `vnom`? Yes, but in voltage_zones

Much of the difficulty arrises from supporting phase-to-phase bounds for both two-phase and three-phase systems. For example, a 2-phase `[a,b]` system only has 1 phase-to-phase bound, whilst a 3-phase system `[a,b,c]` has 3!

A 4-phase system `[a,b,c,d]` is not well-defined; should there be phase-to-phase bounds for each permutation, i.e., `[(a,b), (a,c), (a,d), (b,c), (b,d), (c,d)]`, or only for adjacent ones as it would apply to a 4-phase delta connection, `[(a,b), (b,c), (c,d), (a,d)]`? For three-phase systems these two options coincide.

In order to avoid all of this confusion, we introduce this component, which is at most 3-phase, `|phases|<=3`, and which specifies all bounds symmetrically. For example,
- `phases=[1,2,3], vm_pp_max=440` implies |U1-U2|, |U2-U3|, |U1-U3| <= 440
- `phases=[1,2], vm_pn_max=440` implies |U1-U2| <= 440

This keeps the user from specifying things that do not make sense. This type of bus would suffice for most of the use cases.


| name         | default | type   | description                                               |
| ------------ | ------- | ------ | --------------------------------------------------------- |
| id           | /       |        | unique identifier                                         |
| phases       | [1,2,3] | Vector |                                                           |
| neutral      | 4       |        | maximum conductor-to-ground voltage magnitude             |
| vm_pn_max    | /       | Real   | maximum phase-to-neutral voltage magnitude for all phases |
| vm_pn_min    | /       | Real   | minimum phase-to-neutral voltage magnitude for all phases |
| vm_pp_max    | /       | Real   | maximum phase-to-phase voltage magnitude for all phases   |
| vm_pp_min    | /       | Real   | minimum phase-to-phase voltage magnitude for all phases   |
| vm_ng_rating | /       | Real   | maximum neutral-to-ground voltage magnitude               |

Should we automatically imply vmax and vmin bounds for this?, i.e.
- `vmax = [fill(vm_pn_max+vm_ng_rating, |phases|)..., vm_ng_rating]`
- `vmin = [fill(vm_pn_min-vm_ng_rating, |phases|)..., 0]`

## Line

This is a pi-model branch.
- A linecode implies `rs`, `xs`, `b_fr`, `b_to`, `g_fr` and `g_to`; when these properties are additionally specified, they overwrite the one supplied through the linecode?


| name          | default | type | description                                                              |
| ------------- | ------- | ---- | ------------------------------------------------------------------------ |
| id            | /       |      | unique identifier                                                        |
| f_bus         | /       |      |                                                                          |
| f_connections | /       |      | indicates for each conductor, to which terminal of the f_bus it connects |
| t_bus         | /       |      |                                                                          |
| t_connections | /       |      | indicates for each conductor, to which terminal of the t_bus it connects |
| linecode      | /       |      | a linecode                                                               |
| rs            | /       |      | series resistance matrix, size of n_conductors x n_conductors            |
| xs            | /       |      | series reactance matrix, size of n_conductors x n_conductors             |
| g_fr          | /       |      | from-side conductance                                                    |
| b_fr          | /       |      | from-side susceptance                                                    |
| g_to          | /       |      | to-side conductance                                                      |
| b_to          | /       |      | to-side susceptance                                                      |
| c_rating      | /       |      | symmetrically applicable current rating                                  |
| s_rating      | /       |      | symmetrically applicable power rating                                    |

Add example for linecodes/lines off different size (conductors)

## Linecode

- Should the linecode also include a `c_rating` and `s_rating`? I think not. `c_rating` yes, `s_rating` does not make sense. `s_rating`, if desired, should be specified at the line level
- Does `cmatrix` makes more sense than `g_fr, b_fr, g_to, b_to` when parametrized per length? No, enforce symmetry. If needed place a shunt or inject at low-level.

| name | default | type | description                          |
| ---- | ------- | ---- | ------------------------------------ |
| id   | /       |      | unique identifier                    |
| rs   | /       |      | series resistance matrix             |
| xs   | /       |      | series reactance matrix n_conductors |
| g_fr | /       |      | from-side conductance                |
| b_fr | /       |      | from-side susceptance                |
| g_to | /       |      | to-side conductance                  |
| b_to | /       |      | to-side susceptance                  |

## Shunt

| name        | default | type | description                                                |
| ----------- | ------- | ---- | ---------------------------------------------------------- |
| id          | /       |      | unique identifier                                          |
| bus         | /       |      |                                                            |
| connections | /       |      |                                                            |
| g           | /       |      | conductance, size should be \|connections\|x\|connections\||
| b           | /       |      | susceptance, size should be \|connections\|x\|connections\||

## Capacitor

| name        | default | type | description                                                |
| ----------- | ------- | ---- | ---------------------------------------------------------- |
| id          | /       |      | unique identifier                                          |
| bus         | /       |      |                                                            |
| configuration | /       |      | for delta, 2/3 connections only
| connections | /       |      |                                                            |
| qd_ref      | /       | Vector{Real} | conductance, size should be \|connections\|x\|connections\||
| vnom        | /       | Real     | conductance, size should be \|connections\|x\|connections\||

Give some examples

## Load

| name          | default | type         | description                                                     |
| ------------- | ------- | ------------ | --------------------------------------------------------------- |
| id            | /       |              | unique identifier                                               |
| bus           | /       |              |                                                                 |
| connections   | /       |              |                                                                 |
| configuration | /       | {wye, delta} | if wye-connected, the last connection will indicate the neutral |
| model         | /       |              | indicates the type of voltage-dependency                        |
| pd_ref | /       | Vector{Real} |             |
| qd_ref | /       | Vector{Real} |             |

### `model=constant_current/impedance`

| name   | default | type         | description |
| ------ | ------- | ------------ | ----------- |
| vnom   | /       | Real         |             |

### `model=exponential`

| name   | default | type         | description |
| ------ | ------- | ------------ | ----------- |
| vnom   | /       | Real         |             |
| exp_p  | /       | Real         |             |
| exp_q  | /       | Real         |             |

### `model=ZIP`

| name      | default | type         | description |
| --------- | ------- | ------------ | ----------- |
| vnom      | /       | Real         |             |
| coeff_z_p | /       | Real         |             |
| coeff_i_p | /       | Real         |             |
| coeff_p_p | /       | Real         |             |
| coeff_z_q | /       | Real         |             |
| coeff_i_q | /       | Real         |             |
| coeff_p_q | /       | Real         |             |

Also support this explicitly, or as a separate component? Yes, but not priority

## Generator

| name          | default | type         | description                                                     |
| ------------- | ------- | ------------ | --------------------------------------------------------------- |
| id            | /       |              | unique identifier                                               |
| bus           | /       |              |                                                                 |
| connections   | /       |              |                                                                 |
| configuration | /       | {wye, delta} | if wye-connected, the last connection will indicate the neutral |
| pg_min        | /       |              | lower bound on active power generation per phase                |
| pg_max        | /       |              | upper bound on active power generation per phase                |
| qg_min        | /       |              | lower bound on reactive power generation per phase              |
| qg_max        | /       |              | upper bound on reactive power generation per phase              |

## Assymetric, Lossless, Two-Winding (AL2W) Transformer

These are transformers are assymetric (A), lossless (L) and two-winding (2W). Assymetric refers to the fact that the secondary is always has a `wye` configuration, whilst the primary can be `delta`.

| name          | default                | type         | description                                                              |
| ------------- | ---------------------- | ------------ | ------------------------------------------------------------------------ |
| id            | /                      |              | unique identifier                                                        |
| n_phases      | size(rs)[1]            | Int>0        | number of phases                                                         |
| f_bus         | /                      |              |                                                                          |
| f_connections | /                      |              | indicates for each conductor, to which terminal of the f_bus it connects |
| t_bus         | /                      |              |                                                                          |
| t_connections | /                      |              | indicates for each conductor, to which terminal of the t_bus it connects |
| configuration | /                      | {wye, delta} | for the from-side; the to-side is always connected in wye                |
| tm_nom        | /                      | Real         | nominal tap ratio for the transformer                                    |
| tm_max        | /                      | Vector       | maximum tap ratio for each phase (base=`tm_nom`)                         |
| tm_min        | /                      | Vector       | minimum tap ratio for each phase (base=`tm_nom`)                         |
| tm_set        | fill(1.0, `n_phases`)  | Vector       | set tap ratio for each phase (base=`tm_nom`)                             |
| tm_fix        | fill(true, `n_phases`) | Vector       | indicates for each phase whether the tap ratio is fixed                  |

TODO: add tm stuff

## Transformer

These are n-winding, n-phase, lossy transformers. Note that most properties are now Vectors (or Vectors of Vectors), indexed over the windings.

| name           | default     | type                 | description                                                                                                    |
| -------------- | ----------- | -------------------- | -------------------------------------------------------------------------------------------------------------- |
| id             | /           |                      | unique identifier                                                                                              |
| n_phases       | size(rs)[1] | Int>0                | number of phases                                                                                               |
| n_windings     | size(rs)[1] | Int>0                | number of windings                                                                                             |
| bus            | /           | Vector               | list of bus for each winding                                                                                   |
| connections    |             | Vector{Vector}       | list of connection for each winding                                                                            |
| configurations |             | Vector{{wye, delta}} | list of configuration for each winding                                                                         |
| xsc            | 0.0         | Vector               | list of short-circuit reactances between each pair of windings; enter as a list of the upper-triangle elements |
| rs             | 0.0         | Vector               | list of the winding resistance for each winding                                                                |
| tm_nom         | /           | Vector{Real}         | nominal tap ratio for the transformer                                                                          |
| tm_max         | /           | Vector{Vector}       | maximum tap ratio for each winding and phase (base=`tm_nom`)                                                   |
| tm_min         | /           | Vector{Vector}       | minimum tap ratio for for each winding and phase (base=`tm_nom`)                                               |
| tm_set         | 1.0         | Vector{Vector}       | set tap ratio for each winding and phase (base=`tm_nom`)                                                       |
| tm_fix         | true        | Vector{Vector}       | indicates for each winding and phase whether the tap ratio is fixed                                            |

## Switch/Recloser/Fuse
- Decide which make sense for us. I am inclined to define a switch alone, as these are well-defined in the context of OPF. Other options should be specified by the specific application.
- Ignore loss model for now

| name          | default | type | description                                                              |
| ------------- | ------- | ---- | ------------------------------------------------------------------------ |
| id            | /       |      | unique identifier                                                        |
| state            | /       |      | unique identifier                                                        |
| f_bus         | /       |      |                                                                          |
| f_connections | /       |      | indicates for each conductor, to which terminal of the f_bus it connects |
| t_bus         | /       |      |                                                                          |
| t_connections | /       |      | indicates for each conductor, to which terminal of the t_bus it connects |
| c_rating      | /       |      | symmetrically applicable current rating                                  |
| s_rating      | /       |      | symmetrically applicable power rating                                    |

## Storage

- How to include the inverter model for this? Similar issue as for a PV generator


| name          | default | type         | description                                                     |
| ------------- | ------- | ------------ | --------------------------------------------------------------- |
| id            | /       |              | unique identifier                                               |
| bus           | /       |              |                                                                 |
| connections   | /       |              |                                                                 |
| configuration | /       | {wye, delta} | if wye-connected, the last connection will indicate the neutral |
| energy | |  |  |
| energy_rating | |  |  |
| charge_rating | |  |  |
| discharge_rating | |  |  |
| s_rating | | Vector |  |
| c_rating | | Vector |  |
| charge_efficiency | |  |  |
| discharge_efficiency | |  |  |
| qmax | | Vector |  |
| qmin | | Vector |  |
| r | | Vector |  |
| x | | Vector |  |
| p_loss | |  | total active power standby loss |
| q_loss | |  | total reactive power standby loss |
