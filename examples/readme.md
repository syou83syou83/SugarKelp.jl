![](OceanBioME_headerbar.jpg?raw=true)
### Ocean Biogeochemial Modelling Environment

## Description
OceanBioME was developed with generous support from the Cambridge Centre for Climate Repair [CCRC](https://www.climaterepair.cam.ac.uk) and the Gordon and Betty Moore Foundation as a tool to study the effectiveness and impacts of ocean carbon dioxide removal (CDR) strategies.

OceanBioME is a flexible modelling environment written in Julia for modelling the coupled interactions between ocean biogeochemistry, carbonate chemistry, and physics. OceanBioME can be run as a stand-alone box model, or coupled with Oceananigans.jl to run as a 1D column model or with 3D physics. 

## Installation (before release):

- Pull this repo `git clone --depth 1 USERNAME@github.com:OceanBioME/OceanBioME.jl.git` then move into this directory (i.e. `cd OceanBioME`)
- Activate julia with the `--project` flag pointed to this directory, i.e. `julia --project`, or using `] activate .` from the Julia REPL
- Ensure all the dependencies are installed by typing `] instantiate` or `using Pkg; Pkg,instantiate()` (if the dependencies are not currently listed properly you may have to manually add them)
- Now you can use the package with `using OceanBioME` as usual

> To use the examples you should also clone the [examples data](https://github.com/sichen-org/OceanBioME_example_data) repo into the examples folder by: `cd example; git clone --depth 1 USERNAME@github.com:OceanBioME/OceanBioME_example_data.git`

## Usage overview
OceanBioME can be used as a stand-alone box model or coupled with ocean physics using Oceananigans.jl

### Box model
To set up OceanBioME as a box model, call `Setup.BoxModel` with the biogeochemical model of your choice. For example to use the LOBSTER model:
``` 
model = Setup.BoxModel(:LOBSTER, params, (P=Pᵢ, Z=Zᵢ, D=Dᵢ, DD=DDᵢ, NO₃=NO₃ᵢ, NH₄=NH₄ᵢ, DOM=DOMᵢ), 0.0, 1.0*year)
``` 
Where `Pᵢ` etc. are the initial values. Then call solution = `BoxModel.run(model)` which will give the results and timestamps.

### Defining a model
To setup a new model:
1.  Create a file structured like a module (i.e. starting with `module MODELNAME`  and ending ` end` )
2. In this file write the forcing function with arguments `x, y, z, t` , then ALL core tracers (i.e. even if a function doesn't depend on some fields include them as arguments anyway), and finally `params` 
3. Define a tuple with the core tracers listed called `tracers` 
4. Define a NamedTuple of the forcing functions for each tracer named `forcing_functions` 
5. Define a named tuple `sinking`  where any fields which have a sinking forcing should be listed with a function of `z`  and `params` which calculates their sinking velocity
6. For optional tracer sets (e.g. carbonate chemistry) define the forcing functions as above
7. Define a NamedTuple for the sets of optional tracers named `optional_tracers`, where the optional sets are listed with a tuple of their tracers. E.g. `optional_tracers=(carbonates=(:DIC, :ALK), oxygen=(:OXY,))`
8. If you are NOT defining any optional sets you must still define the optional tracers tuple but leave it empty (i.e. `optional_tracers=NameTuple()`), and the same for sinking
9. Finally it is most likely useful to define the default parameters, hese can be listed in a separate file and included or listed directly as a NamedTuple. So far the default set has been named `default`     

See the LOBSTER model for how functional PAR dependance can be dealt with.

### Oceananigans

1. Define the physical dependencies: `T`, `S`, and `PAR`. Currently, these can be functions of `(x, y, z, t)`, or, for `T` and `S` tracer fields (performs much slower). Additionally, `PAR` can be an auxiliary field calculated by callbacks provided by OceanBioME.jl. For example:
```
t_function(x, y, z, t) = temperature_itp(mod(t, 364days)) .+ 273.15
s_function(x, y, z, t) = salinity_itp(mod(t, 364days))
surface_PAR(t) = surface_PAR_itp(mod(t, 364days))

PAR = Oceananigans.Fields.Field{Center, Center, Center}(grid)
```
This latter example requires that the grid is already defined.

2. Pass these fields and the grid to a setup function (currently Lobster is the only one implimented)
```
bgc = Lobster.setup(grid, params, (T=t_function, S=s_function, PAR=PAR))
```
This returns a model with fields `tracers`, `forcing`, and `boundary_conditions`.

3. Pass the result to an Oceananigans model (along with the auxiliary fields if necessary)
```
model = NonhydrostaticModel(...
                            tracers = (:b, bgc.tracers...),
                            ...
                            forcing =  bgc.forcing,
                            boundary_conditions = merge((u=u_bcs, b=buoyancy_bcs), bgc.boundary_conditions),
                            auxiliary_fields = (PAR=PAR, ))
```

4. Finally, once you have otherwise defined the simulation, setup the callback to update the PAR (here Light.update_2λ! is a function provided by OceanBioME.jl that requires a surface PAR interpolant be passed to it. (At some point this will need to be updated to be an x, y, t interpolation).
```
simulation.callbacks[:update_par] = Callback(Light.update_2λ!, IterationInterval(1), merge(params, (surface_PAR=surface_PAR,)))

```

### Particles
To include particles which interact with the BGC model setup the model as above, but before the Oceananigans model is defined set up the particles by:

1. Define a function of `(x, y, z, t, property dependencies..., params, Δt)`, for example:
```
function sugarkelpequations(x, y, z, t, A, N, C, NO₃, irr, params, Δt)
  dA, dN, dC, j = SugarKelp.equations(A, N, C, params.urel, params.temp(mod(t, 364days), 
                                                irr, NO₃, params.λ[1+floor(Int, mod(t, 364days)/day)], params.resp_model, Δt, params.paramset)
  return (A = dA / (60*60*24), N = dN / (60*60*24), C = dC / (60*60*24), j = (A+Δt*dA / (60*60*24)) * j / (60*60*24*14*0.001))#fix units
end
lat=57.5
λ_arr=SugarKelp.gen_λ(lat)
```
2. Next create the particle structure as is done with Oceananigans
```
struct Kelp
  #position
  x :: AbstractFloat
  y :: AbstractFloat
  z :: AbstractFloat
  #velocity
  u :: AbstractFloat
  v :: AbstractFloat
  w :: AbstractFloat
  #properties
  A :: AbstractFloat
  N :: AbstractFloat
  C :: AbstractFloat
  j :: AbstractFloat
  #tracked fields
  NO₃  :: AbstractFloat
  PAR :: AbstractFloat
end
... #define initial value arrays
particles = StructArray{Kelp}((x₀ₖ, y₀ₖ, z₀ₖ, u₀ₖ, v₀ₖ, w₀ₖ, a₀, n₀, c₀, j₀, NO₃₀, PAR₀))
```
3. Define the "source fields" which are those tracked by the particle (usually arguments of the function), and "sink fields" which are the connection between particle properties and tracers
```
source_fields = ((tracer=:NO₃, property=:NO₃, scalefactor=1.0), (tracer=:PAR, property=:PAR, scalefactor=1.0))
sink_fields = ((tracer=:NO₃, property=:j, scalefactor=-1.0), )
``` 
4. Call the particle setup function to define Oceananigans LagrangianParticles
``` 
kelp_particles = Particles.setup(particles, sugarkelpequations, 
                                        (:A, :N, :C, :NO₃, :PAR), #forcing function property dependencies
                                        (urel=0.15, temp=temperature_itp, λ=λ_arr, resp_model=2, paramset=SugarKelp.broch2013params), #forcing function parameters
                                        (:A, :N, :C), #forcing function integrals
                                        (:j, ), #forcing function diagnostic fields
                                        source_fields,
                                        sink_fields,
                                        100.0)#density
``` 
5. Finally, setup the model with the particles
``` 
model = NonhydrostaticModel(advection = UpwindBiasedFifthOrder(),
                            timestepper = :RungeKutta3,
                            grid = grd,
                            tracers = (:b, bgc.tracers...),
                            coriolis = FPlane(f=1e-4),
                            buoyancy = BuoyancyTracer(), 
                            closure = ScalarDiffusivity(ν=κₜ, κ=κₜ), 
                            forcing =  bgc.forcing,
                            boundary_conditions = merge((u=u_bcs, b=buoyancy_bcs), bgc.boundary_conditions),
                            auxiliary_fields = (PAR=PAR, ),
                            particles = kelp_particles)
``` 

### Sediment model
Sediment uses a simple model where the concentration of two types of organic matter with different decomposition rates are tracked at the bottom of the domain, and provide a boundary condition on the detritus, inorganic nitrates, and carbonates. To use:
1. Setup the boundary conditions
``` 
sediment_bcs=Boundaries.setupsediment(grid)
``` 
2. Add these in the BGC model definition, and the auxillary fields to the model definition
``` 
bgc = Setup.Oceananigans(:LOBSTER, grid, params, PAR_field, optional_sets=(:carbonates, :oxygen), topboundaries=(DIC=dic_bc, OXY=oxy_bc), bottomboundaries = sediment_bcs.boundary_conditions)
...
model = NonhydrostaticModel(advection = UpwindBiasedFifthOrder(),
                            timestepper = :RungeKutta3,
                            grid = grid,
                            tracers = (:b, bgc.tracers...),
                            coriolis = FPlane(f=1e-4),
                            buoyancy = BuoyancyTracer(), 
                            closure = ScalarDiffusivity(ν=κₜ, κ=κₜ), 
                            forcing =  bgc.forcing,
                            boundary_conditions = bgc.boundary_conditions,
                            auxiliary_fields = merge((PAR=PAR_field, ), sediment_bcs.auxiliary_fields)  #comment out this line if using functional form of PAR
                            )
``` 
3. Run as normal

## Plotting functionality
Some plotting utilities are provided.
