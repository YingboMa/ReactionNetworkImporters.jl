# ReactionNetworkImporters.jl

[![Build status](https://ci.appveyor.com/api/projects/status/wqq5flk2w8asad78/branch/master?svg=true)](https://ci.appveyor.com/project/isaacsas/reactionnetworkimporters-jl/branch/master)

This package provides importers to load reaction networks from several file formats. Currently it supports loading networks in the following formats:
1. A *subset* of the BioNetGen .net file format.
2. The basic format used by the [RSSA](https://www.cosbi.eu/research/prototypes/rssa) group at COSBI in their [model collection](https://www.cosbi.eu/prototypes/jLiexDeBIgFV4zxwnKiW97oc4BjTtIoRGajqdUz4.zip).

Imported networks are currently output as a [DiffEqBiological](https://github.com/JuliaDiffEq/DiffEqBiological.jl/), `min_reaction_network`.

----
## Examples

### Loading a BioNetGen .net file
A simple network from the builtin BioNetGen bngl examples is the [repressilator](data/repressilator/Repressilator.bngl). The `generate_network` command in the bngl file outputs a reduced network description, i.e. a [.net](data/repressilator/Repressilator.net) file, which can be loaded into a DiffEqBiological `min_reaction_network` as:
```julia
using ReactionNetworkImporters
fname = "PATH/TO/Repressilator.net"
prnbng = loadrxnetwork(BNGNetwork(), "BNGRepressilator", fname)
```
Here `BNGNetwork` is a type specifying the file format that is being loaded, and `BNGRepressilator` specifies the type of the generated `min_reaction_network`, see [DiffEqBiological](https://github.com/JuliaDiffEq/DiffEqBiological.jl/). `prnbng` is a `ParsedReactionNetwork` structure with the following fields:
- `rn`, a DiffEqBiological `min_reaction_network`
- `u₀`, the initial condition (as a `Vector{Float64}`)
- `p`, the parameter vector (as a `Vector{Float64}`)
- `paramexprs`, the parameter vector as a mix of `Numbers`, `Symbols` and `Exprs`. `p` is generated by evaluation of these expressions and symbols.
- `symstonames`, a `Dict` mapping from the internal `Symbol` of a species used in the generated `min_reaction_network` to a `Symbol` generated from the name in the .net file. This is necessary as BioNetGen can generate exceptionally long species names, involving characters that lead to malformed species names when used with `DiffEqBiological`.
- `groupstoids`, a `Dict` mapping the `Symbols` (i.e. names) for any species groups defined in the .net file to a vector of indices into `u₀` where the corresponding species are stored.
- `rnstr`, a string representation of the full DiffEqBiological DSL command that was evaluated to generate the network.

Given `prnbng`, we can construct and solve the corresponding ODE model for the reaction system by
```julia
using OrdinaryDiffEq, DiffEqBiological
rn = prnbng.rn
addodes!(rn)
tf = 100000.0
oprob = ODEProblem(rn, prnbng.u₀, (0.,tf), prnbng.p)
sol = solve(oprob, Tsit5(), saveat=tf/1000.)
```
See the [DiffEqBiological documentation](https://github.com/JuliaDiffEq/DiffEqBiological.jl/) for how to generate ODE, SDE, jump and other types of models.

### Loading a RSSA format network file
As the licensing is unclear we can not redistribute any example RSSA formatted networks. They can be downloaded from the model collection link listed above. Assuming you've saved both a reaction network file and corresponding initial condition file, they can be loaded as
```julia
initialconditionf = "PATH/TO/FILE"
networkf = "PATH/TO/FILE"
rssarn = loadrxnetwork(RSSANetwork(), "RSSARxSys", initialconditionf, networkf)
```
Here `RSSANetwork` specifies the type of the file to parse, and `RSSARxSys` gives the type of the generated `min_reaction_network`. `rssarn` is again a `ParsedReactionNetwork`, but only the `rn`, `u₀` and `rnstr` fields will now be relevant (the remaining fields will be set to `nothing`).
