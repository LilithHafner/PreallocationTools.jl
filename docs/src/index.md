# PreallocationTools.jl

PreallocationTools.jl is a set of tools for helping build non-allocating
pre-cached functions for high-performance computing in Julia. Its tools handle
edge cases of automatic differentiation to make it easier for users to get
high performance even in the cases where code generation may change the
function that is being called.

## DiffCache

`DiffCache` is a method for generating doubly-preallocated vectors which are
compatible with non-allocating forward-mode automatic differentiation by
ForwardDiff.jl. Since ForwardDiff uses chunked duals in its forward pass, two
vector sizes are required in order for the arrays to be properly defined.
`DiffCache` creates a dispatching type to solve this, so that by passing a
qualifier it can automatically switch between the required cache. This method
is fully type-stable and non-dynamic, made for when the highest performance is
needed.

### Using DiffCache

```julia
DiffCache(u::AbstractArray, N::Int=ForwardDiff.pickchunksize(length(u)); levels::Int = 1)
DiffCache(u::AbstractArray, N::AbstractArray{<:Int})
```

The `DiffCache` function builds a `DiffCache` object that stores both a version
of the cache for `u` and for the `Dual` version of `u`, allowing use of
pre-cached vectors with forward-mode automatic differentiation. Note that
`DiffCache`, due to its design, is only compatible with arrays that contain concretely
typed elements.

To access the caches, one uses:

```julia
get_tmp(tmp::DiffCache, u)
```

When `u` has an element subtype of `Dual` numbers, then it returns the `Dual`
version of the cache. Otherwise it returns the standard cache (for use in the
calls without automatic differentiation).

In order to preallocate to the right size, the `DiffCache` needs to be specified
to have the correct `N` matching the chunk size of the dual numbers or larger.
If the chunk size `N` specified is too large, `get_tmp` will automatically resize
when dispatching; this remains type-stable and non-allocating, but comes at the
expense of additional memory.

In a differential equation, optimization, etc., the default chunk size is computed
from the state vector `u`, and thus if one creates the `DiffCache` via
`DiffCache(u)` it will match the default chunking of the solver libraries.

`DiffCache` is also compatible with nested automatic differentiation calls through
the `levels` keyword (`N` for each level computed using based on the size of the
state vector) or by specifying `N` as an array of integers of chunk sizes, which
enables full control of chunk sizes on all differentation levels.

### DiffCache Example 1: Direct Usage

```julia
using ForwardDiff, PreallocationTools
randmat = rand(5, 3)
sto = similar(randmat)
stod = DiffCache(sto)

function claytonsample!(sto, τ, α; randmat=randmat)
    sto = get_tmp(sto, τ)
    sto .= randmat
    τ == 0 && return sto

    n = size(sto, 1)
    for i in 1:n
        v = sto[i, 2]
        u = sto[i, 1]
        sto[i, 1] = (1 - u^(-τ) + u^(-τ)*v^(-(τ/(1 + τ))))^(-1/τ)*α
        sto[i, 2] = (1 - u^(-τ) + u^(-τ)*v^(-(τ/(1 + τ))))^(-1/τ)
    end
    return sto
end

ForwardDiff.derivative(τ -> claytonsample!(stod, τ, 0.0), 0.3)
ForwardDiff.jacobian(x -> claytonsample!(stod, x[1], x[2]), [0.3; 0.0])
```

In the above, the chunk size of the dual numbers has been selected based on the size
of `randmat`, resulting in a chunk size of 8 in this case. However, since the derivative
is calculated with respect to τ and the Jacobian is calculated with respect to τ and α,
specifying the `DiffCache` with `stod = DiffCache(sto, 1)` or `stod = DiffCache(sto, 2)`,
respectively, would have been the most memory efficient way of performing these calculations
(only really relevant for much larger problems).

### DiffCache Example 2: ODEs

```julia
using LinearAlgebra, OrdinaryDiffEq
function foo(du, u, (A, tmp), t)
    mul!(tmp, A, u)
    @. du = u + tmp
    nothing
end
prob = ODEProblem(foo, ones(5, 5), (0., 1.0), (ones(5,5), zeros(5,5)))
solve(prob, TRBDF2())
```

fails because `tmp` is only real numbers, but during automatic differentiation
we need `tmp` to be a cache of dual numbers. Since `u` is the value that will
have the dual numbers, we dispatch based on that:

```julia
using LinearAlgebra, OrdinaryDiffEq, PreallocationTools
function foo(du, u, (A, tmp), t)
    tmp = get_tmp(tmp, u)
    mul!(tmp, A, u)
    @. du = u + tmp
    nothing
end
chunk_size = 5
prob = ODEProblem(foo, ones(5, 5), (0., 1.0), (ones(5,5), DiffCache(zeros(5,5), chunk_size)))
solve(prob, TRBDF2(chunk_size=chunk_size))
```

or just using the default chunking:

```julia
using LinearAlgebra, OrdinaryDiffEq, PreallocationTools
function foo(du, u, (A, tmp), t)
    tmp = get_tmp(tmp, u)
    mul!(tmp, A, u)
    @. du = u + tmp
    nothing
end
chunk_size = 5
prob = ODEProblem(foo, ones(5, 5), (0., 1.0), (ones(5,5), DiffCache(zeros(5,5))))
solve(prob, TRBDF2())
```
### DiffCache Example 3: Nested AD calls in an optimization problem involving a Hessian matrix

```julia
using LinearAlgebra, OrdinaryDiffEq, PreallocationTools, Optim, Optimization
function foo(du, u, p, t)
    tmp = p[2]
    A = reshape(p[1], size(tmp.du))
    tmp = get_tmp(tmp, u)
    mul!(tmp, A, u)
    @. du = u + tmp
    nothing
end

coeffs = -collect(0.1:0.1:0.4)
cache = DiffCache(zeros(2,2), levels = 3)
prob = ODEProblem(foo, ones(2, 2), (0., 1.0), (coeffs, cache))
realsol = solve(prob, TRBDF2(), saveat = 0.0:0.1:10.0, reltol = 1e-8)

function objfun(x, prob, realsol, cache)
    prob = remake(prob, u0 = eltype(x).(prob.u0), p = (x, cache))
    sol = solve(prob, TRBDF2(), saveat = 0.0:0.1:10.0, reltol = 1e-8)

    ofv = 0.0
    if any((s.retcode != :Success for s in sol))
        ofv = 1e12
    else
        ofv = sum((sol.-realsol).^2)
    end
    return ofv
end
fn(x,p) = objfun(x, p[1], p[2], p[3])
optfun = OptimizationFunction(fn, Optimization.AutoForwardDiff())
optprob = OptimizationProblem(optfun, zeros(length(coeffs)), (prob, realsol, cache))
solve(optprob, Newton())
```
Solves an optimization problem for the coefficients, `coeffs`, appearing in a differential equation.
The optimization is done with [Optim.jl](https://github.com/JuliaNLSolvers/Optim.jl)'s `Newton()`
algorithm. Since this involves automatic differentiation in the ODE solver and the calculation
of Hessians, three automatic differentiations are nested within each other. Therefore, the `DiffCache`
is specified with `levels = 3`.

## FixedSizeDiffCache

`FixedSizeDiffCache` is a lot like `DiffCache`, but it stores dual numbers in its caches
instead of a flat array. Because of this, it can avoid a view, making it a little bit
more performant for generating caches of non-`Array` types. However, it is a lot less
flexible than `DiffCache`, and is thus only recommended for cases where the chunk size
is known in advance (for example, ODE solvers) and where `u` is not an `Array`.

The interface is almost exactly the same, except with the constructor:

```julia
FixedSizeDiffCache(u::AbstractArray, chunk_size = Val{ForwardDiff.pickchunksize(length(u))})
FixedSizeDiffCache(u::AbstractArray, chunk_size::Integer)
```

Note that the `FixedSizeDiffCache` can support duals that are of a small chunk size than
the preallocated ones, but not a larger size. Nested duals are not supported with this
construct.

## LazyBufferCache

```julia
LazyBufferCache(f::F=identity)
```

A `LazyBufferCache` is a `Dict`-like type for the caches which automatically defines
new cache arrays on demand when they are required. The function `f` maps
`size_of_cache = f(size(u))`, which by default creates cache arrays of the same size.

Note that `LazyBufferCache` does cause a dynamic dispatch, though it is type-stable.
This gives it a ~100ns overhead, and thus on very small problems it can reduce
performance, but for any sufficiently sized calculation (e.g. >20 ODEs) this
may not be even measurable. The upside of `LazyBufferCache` is that the user does
not have to worry about potential issues with chunk sizes and such: `LazyBufferCache`
is much easier!

### Example

```julia
using LinearAlgebra, OrdinaryDiffEq, PreallocationTools
function foo(du, u, (A, lbc), t)
    tmp = lbc[u]
    mul!(tmp, A, u)
    @. du = u + tmp
    nothing
end
prob = ODEProblem(foo, ones(5, 5), (0., 1.0), (ones(5,5), LazyBufferCache()))
solve(prob, TRBDF2())
```

## Note About ReverseDiff Support for LazyBuffer

ReverseDiff support is done in SciMLSensitivity.jl to reduce the AD requirements on this package.
Load that package if ReverseDiff overloads are required.

## Similar Projects

[AutoPreallocation.jl](https://github.com/oxinabox/AutoPreallocation.jl) tries
to do this automatically at the compiler level. [Alloc.jl](https://github.com/FluxML/Alloc.jl)
tries to do this with a bump allocator.

## Contributing

- Please refer to the
  [SciML ColPrac: Contributor's Guide on Collaborative Practices for Community Packages](https://github.com/SciML/ColPrac/blob/master/README.md)
  for guidance on PRs, issues, and other matters relating to contributing to SciML.
- See the [SciML Style Guide](https://github.com/SciML/SciMLStyle) for common coding practices and other style decisions.
- There are a few community forums:
    - The #diffeq-bridged and #sciml-bridged channels in the
      [Julia Slack](https://julialang.org/slack/)
    - The #diffeq-bridged and #sciml-bridged channels in the
      [Julia Zulip](https://julialang.zulipchat.com/#narrow/stream/279055-sciml-bridged)
    - On the [Julia Discourse forums](https://discourse.julialang.org)
    - See also [SciML Community page](https://sciml.ai/community/)


## Reproducibility
```@raw html
<details><summary>The documentation of this SciML package was built using these direct dependencies,</summary>
```
```@example
using Pkg # hide
Pkg.status() # hide
```
```@raw html
</details>
```
```@raw html
<details><summary>and using this machine and Julia version.</summary>
```
```@example
using InteractiveUtils # hide
versioninfo() # hide
```
```@raw html
</details>
```
```@raw html
<details><summary>A more complete overview of all dependencies and their versions is also provided.</summary>
```
```@example
using Pkg # hide
Pkg.status(;mode = PKGMODE_MANIFEST) # hide
```
```@raw html
</details>
```
```@raw html
You can also download the
<a href="
```
```@eval
using TOML
version = TOML.parse(read("../../Project.toml",String))["version"]
name = TOML.parse(read("../../Project.toml",String))["name"]
link = "https://github.com/SciML/"*name*".jl/tree/gh-pages/v"*version*"/assets/Manifest.toml"
```
```@raw html
">manifest</a> file and the
<a href="
```
```@eval
using TOML
version = TOML.parse(read("../../Project.toml",String))["version"]
name = TOML.parse(read("../../Project.toml",String))["name"]
link = "https://github.com/SciML/"*name*".jl/tree/gh-pages/v"*version*"/assets/Project.toml"
```
```@raw html
">project</a> file.
```