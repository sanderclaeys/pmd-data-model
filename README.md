# pmd-data-model
draft of data model for PowerModelsDistribution.jl

For a draft of our high-level (engineering) data model for PowerModelsDistribution.jl, please see [data_model_eng.md](data_model_eng.md). 
We would greatly appreciate your feedback; you can simply open an issue where you can raise
- something that is unclear;
- a use case you have that is missing;
- ...

### Why are we doing this?
In short, this data model extends a subset of the OpenDSS data model with properties specific to optimization, such as
- technical constraints (voltage bounds, current bounds...),
- generation costs,
- ...

This will allow us to *specify test cases for distribution network optimization in a well-defined way*. 

In the past, we parsed OpenDSS data directly to a low-level, mathematical representation of the network. 
This made it hard to preprocess the data, as much high-level information was lost in the process.
Also, it was difficult to inspect what the parser was reading in, because it was immediately transformed/mapped in various ways.
We aim to *make PowerModelsDistribution.jl more user-friendly by adding this high-level data model*.


