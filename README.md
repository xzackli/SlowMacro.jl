# ReferenceImplementations.jl

<!-- [![Stable](https://img.shields.io/badge/docs-stable-blue.svg)](https://xzackli.github.io/ReferenceImplementations.jl/stable) -->
[![Dev](https://img.shields.io/badge/docs-dev-blue.svg)](https://xzackli.github.io/ReferenceImplementations.jl/dev)
[![Build Status](https://github.com/xzackli/ReferenceImplementations.jl/workflows/CI/badge.svg)](https://github.com/xzackli/ReferenceImplementations.jl/actions)
[![codecov](https://codecov.io/gh/xzackli/ReferenceImplementations.jl/branch/main/graph/badge.svg?token=rM1AU0MQ38)](https://codecov.io/gh/xzackli/ReferenceImplementations.jl)

This package exports `@slowdef` and `@slow` macros to help you write fast scientific code. The `@slow` macro applies a [Cassette](https://github.com/JuliaLabs/Cassette.jl) pass to each 
top-level function in the input expression, recursively replacing nested methods that have alternative implementations provided by `@slowdef`.
A single function can be replaced via `@slow f (expression)`. 

For instructions, please consult the [documentation](https://xzackli.github.io/ReferenceImplementations.jl/dev).


## Examples

Calling `@slow` on an expression calls every method with a slow implementation
in the nested sequence of calls for that expression.

```julia
using ReferenceImplementations
@slowdef mysin(x) = begin println("slow mysin"); return sin(x) end
mysin(x) = begin println("fast mysin"); return sin(x) end

# call the slow version
@slow mysin(0.)  # prints "slow mysin"
mysin(0.)        # prints "fast mysin"
```

This works for `@slowdef` functions that are nested inside other functions in the expression.

```julia
@slowdef f(x) = begin println("slow f"); return mysin(x)^2 end
f(x) = begin println("fast f"); return mysin(x)^2 end

# call the slow version
@slow f(0.)  # prints "slow f", "slow mysin"
f(0.)        # prints "fast f", "fast mysin"
```

You can target individual functions for slowing by passing a function after slow.

```julia
@slow mysin f(0.)  # prints "fast f", "slow mysin"
@slow f f(0.)  # prints "slow f", "fast mysin"
```

Using `@slow` does incur some compilation cost, but subsequent calls should be fast.

## Why?

I often write two versions of a function,

* **V1: Naive implementation.** Since Julia is so expressive, this implementation is usually short and resembles the published equations or pseudocode.
* **V2: Optimized implementation.** This version is written for a computer, i.e. ⊂ { exploits symmetries, reuses allocated memory, hits the cache in a friendly way, reorders calculations for SIMD, divides the work with threads, precomputes parts, caches intermediate expressions, ... }.

V1 is easier to understand and extend. V2 is the implementation exported in your package and it's often much faster, but complicated and verbose. Julia sometimes allows you to use abstractions such that V1 ≈ V2, but this is not always possible. ReferenceImplementations.jl lets you keep both.

## How?

`@slowdef` injects a first argument into the method signature, doing the transform
```julia
func(args...; kwargs...)  ⇨  func(::ReferenceImplementations.RefImpl, args...; kwargs...)
``` 
with the same type signatures (preserving `where` and `::T`, for example). The `@slow` macro then applies a Cassette pass for each top-level function call in an expression which replaces `func(args...; kwargs...)` with `func(::ReferenceImplementations.RefImpl, args...; kwargs...)` if that method exists.
