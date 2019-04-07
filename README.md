# LinearDynamicsModels.jl

[![Build Status](https://travis-ci.org/schmrlng/LinearDynamicsModels.jl.svg?branch=master)](https://travis-ci.org/schmrlng/LinearDynamicsModels.jl)
[![Build status](https://ci.appveyor.com/api/projects/status/qijrn9gssps4t8hx?svg=true)](https://ci.appveyor.com/project/schmrlng/lineardynamicsmodels-jl)
[![codecov.io](http://codecov.io/github/schmrlng/LinearDynamicsModels.jl/coverage.svg?branch=master)](http://codecov.io/github/schmrlng/LinearDynamicsModels.jl?branch=master)

This package serves two purposes:

1. To extend the interfaces defined in [`DifferentialDynamicsModels.jl`](https://github.com/schmrlng/DifferentialDynamicsModels.jl) to linear time-invariant systems of the form ![linear dynamics](https://latex.codecogs.com/png.latex?%5Cinline%20f%28x%2C%20u%29%20%3D%20Ax%20&plus;%20Bu%20&plus;%20c) and implement fast solutions to two-point boundary value problems with these dynamics (provided they are controllable), minimizing the mixed time/control effort criterion ![time plus quadratic control](https://latex.codecogs.com/svg.latex?%5Cinline%20%5Cint_0%5ET%20%281%20&plus;%20u%28t%29%5ET%20R%20u%28t%29%29%20%5Cmathop%7B%7D%5C%21%5Cmathrm%7Bd%7Dt) (where ![R](https://latex.codecogs.com/gif.latex?%5Cinline%20R) is symmetric positive definite).
    - `LinearDynamics{Dx,Du} <: DifferentialDynamics` is the main type exported by this package. The type parameters `Dx` and `Du` denote the state and control dimension respectively. [Statically sized arrays](https://github.com/JuliaArrays/StaticArrays.jl) are used in this package for their performance benefits; the type constructor requires arguments of the form `LinearDynamics(A::StaticMatrix{Dx,Dx}, B::StaticMatrix{Dx,Du}, c::StaticVector{Du})`. Though `LinearDynamics` supports arbitrary values for `A`, `B`, and `c`, this package also exports the convenience constructors `DoubleIntegatorDynamics(D::Int)`, `TripleIntegatorDynamics(D::Int)`, and `NIntegratorDynamics(N::Int, D::Int)` where `D` is the spatial dimension (e.g., `DoubleIntegatorDynamics(3)` will model a point mass in three dimensions under controlled acceleration).
    - `LinearQuadraticSteering` is a type alias for a particular parameterization of `SteeringBVP`:
        ```julia
        const LinearQuadraticSteering{Dx,Du,Cache} = SteeringBVP{<:LinearDynamics{Dx,Du},<:TimePlusQuadraticControl{Du},EmptySteeringConstraints,Cache}
        ```
        Depending on whether `SteeringBVP(f::LinearDynamics, j::TimePlusQuadraticControl)` is called with the keyword argument `compile=Val(true)` or `compile=Val(false)` (the default), the resulting `SteeringBVP` instance may contain a cache of optimal control functions/quantities symbolically computed using [SymPy.jl](https://github.com/JuliaPy/SymPy.jl). Compilation greatly reduces BVP computation time (useful if you need to solve millions or even billions of steering problems, as in sampling-based robot motion planning) but introduces a large initial overhead (i.e., stick to `compile=Val(false)` if you only need to solve a few instances of a particular steering setup). Note that for BVP compilation the user must first `using SymPy` or `import SymPy`.

2. To implement functions for dynamics linearization, leveraging automatic differentiation provided by [ForwardDiff.jl](https://github.com/JuliaDiff/ForwardDiff.jl). In particular this package provides linearization of continuous-time systems as well as linearization of the corresponding discrete-time systems arising from zero-order hold or first-order hold input.
    - `linearize(f::DifferentialDynamics, x, u)` — linearization of a differential dynamics model `f` about the state `x` and control `u`; returns a [`LinearDynamics`](https://github.com/schmrlng/LinearDynamicsModels.jl/blob/master/src/LinearDynamicsModels.jl#L29).
    - `linearize(f::DifferentialDynamics, x, u::StepControl)` — linearization of the discrete time model produced by integrating `f` starting from the state `x` and applying the zero-order hold control interval `u` (constant control `u.u` over duration `u.t`); returns a [`ZeroOrderHoldLinearization`](https://github.com/schmrlng/LinearDynamicsModels.jl/blob/master/src/linearization.jl#L109). This linearization is exact (up to numerical error) if `f isa LinearDynamics`.
    - `linearize(f::DifferentialDynamics, x, u::RampControl)` — linearization of the discrete time model produced by integrating `f` starting from the state `x` and applying the first-order hold control interval `u` (control linearly interpolated from `u.u0` to `u.uf` over duration `u.t`); returns a [`FirstOrderHoldLinearization`](https://github.com/schmrlng/LinearDynamicsModels.jl/blob/master/src/linearization.jl#L113). This linearization is exact (up to numerical error) if `f isa LinearDynamics`.