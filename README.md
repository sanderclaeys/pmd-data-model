# PowerModelsDistribution Engineering Data Model

This repository contains the current draft of data model for PowerModelsDistribution.jl

For a draft of our high-level (`"engineering"`) data model for PowerModelsDistribution.jl, please see [data_model_eng.md](data_model_eng.md).

## Why are we doing this?

The primarily reason to change the data model is to adequately support the typical components in distribution systems it was necessary in many cases to *decompose* components into multiple base components that could more easily be represented mathematically in *e.g.* the PowerModels data format. The most obvious example of this are distribution level transformers. In order to support n-winding, multiphase transformers a decomposition scheme was created to turn those components into series of ideal 2-winding lossless transformers and impedance branches. To connect these decomposed components together, virtual buses also have to be introduced. The result of this is a data model that does not directly represent the original network topologically. This has also created difficulty in applying preprocessing to networks, since high-level information was lost in the process of parsing in the files. In this new schema, preprocessing to *e.g.* remove small impedances, perform Kron reductions, or even convert transformers into impedance branches, etc., should be much simpler.

In addition, because the common data format that we use for distribution networks, OpenDSS, supports arbitrary string names for components, as opposed to integers as we have traditionally supported, many users commented that debugging their networks once parsed into PowerModelsDistribution was challenging, and has proven to be a barrier to entry for PowerModelsDistribution. To address this, we have created a schema where the dictionary keys of the components match their names as best as possible in the originating data format.

Finally, although OpenDSS supports a large number of ways to define networks, and can be quite powerful, it is geared towards power flow and harmonics analysis, whereas PowerModelsDistribution is meant to focus on optimization. OpenDSS unfortunately does not have a clean way to define optimization parameters, such as cost models for generators, or per phase bounds (*e.g.* current, voltage, etc.). There are also several features that OpenDSS does not support in regards to transformer definition, such as per phase, per winding tap definitions, which is a feature that has been requested by the user community. This data model will allow us to *specify test cases for distribution network optimization in a well-defined way*.

By adding this high-level data model, dubbed the `"engineering"` data model, to PowerModelsDistribution we aim to significantly increase the user-friendliness of our software.

## What is happening to the old data model?

The old data model isn't really going anywhere, this engineering data model is merely a user-facing interface to the data that should be much easier to work with. The engineering model will still need to be converted into the low-level (`"mathematical"`) data model, but we have moved this step to the solver routines to make things easier for the end user. This logic is contained in the PowerModelsDistribution repository under the `data-model` branch, where active development on this new model is being performed. We still provide easy methods for users to convert the model into the mathematical model as well, so users should not worry about losing any existing functionality, this is merely an abstraction to aid in the creation, maintenance, and analysis of distribution network data. The mechanics of the usage of the engineering vs mathematical model in PowerModelsDistribution will be demonstrated in the documentation of the refactor.

## We want your feedback

As users of PowerModelDistribution, your feedback on this data model is immensely valuable. Because we want to increase the user-friendliness and reduce the barrier to entry for PowerModelsDistribution, incorporating feedback from those working in both power engineering and optimization is desirable.

We encourage you to submit your feedback via GitHub Issues on this repository and we will do our best to incorporate those comments and suggestions. For example, please submit an Issues if there is:

- something that is unclear;
- a use case you have that is missing;
- the naming of a component or parameter is too out-of-conformation compared to current usage;
- ...
