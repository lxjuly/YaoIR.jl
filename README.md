# YaoIR

[![Build Status](https://travis-ci.com/QuantumBFS/YaoIR.jl.svg?branch=master)](https://travis-ci.com/QuantumBFS/YaoIR.jl)
[![Coveralls](https://coveralls.io/repos/github/QuantumBFS/YaoIR.jl/badge.svg?branch=master)](https://coveralls.io/github/QuantumBFS/YaoIR.jl?branch=master)

## Introduction

**Warning: This package is still in early development, I make it public just to enable CI etc. don't use it**

YaoIR is an Intermediate Representation built based on
Julia builtin expression with extended semantic on quantum control, measure and position. Its (extended) syntax is very simple:

### Semantics

The semantic of YaoIR tries to make use of Julia semantic as much as possible so you don't feel this
is not Julian. But since the quantum circuit has some
special semantic that Julia expression cannot express
directly, the semantic of Julia expression is extended in YaoIR.

The point of this new IR is it make use of Julia native
control flow directly instead of unroll the loop and conditions into a Julia type, such as `Chain`, `Kron`,
`ConditionBlock` in QBIR, which improves the performance and provide possibility of further compiler
optimization by analysis done on quantum circuit and classical control flows.

#### Gate Position
gate positions are specific with `=>` at each line,
the `=>` operator inside function calls will not be
parsed, e.g


```jl
1 => H # apply Hadamard gate on the 1st qubit
foo(1=>H) # it means normal Julia pair
1=>foo(x, y, z) # it will parse foo(x, y, z) as a quantum gate/circuit, but will error later if type inference finds they are not.
```

all the gate or circuit's position should be specified by its complete locations, e.g

```jl
1:n => qft(n) # right
1 => qft(n) # wrong
```

but single qubit gates can use multi-location argument
to represent repeated locations, e.g

```jl
1:n => H # apply H on 1:n locations
```

#### Control

`control` is parsed as a special reserved function (means you cannot overload it) in each program, like QBIR, its first argument is the control
location with signs as control configurations and the second argument is a normal gate position argument introduce above.

#### Measure

`measure` is another reserved special function parsed that has specific semantic in the IR (measure the locations passed to it).

### Usage

using it is pretty simple, just use `@device` macro to annotate a "device" function, like CUDA programming, this device function should not return anything but `nothing`.

The compiler will compile this function definition to
a generic circuit `Circuit` with the same name. A generic circuit is a generic quantum program that can
be overload with different Julia types, e.g

```jl
@device function qft(l::Int, n::Int)
    l => H
    for k in l:n
        control(k, l=>Shift(2π/2^(k-l)))
    end

    if n > l
        l+1:n => qft(l+1, n)
    end
end

@device qft(n::Int) = 1:n => qft(1, n)
```

note: all the quantum gates should be annotate with its corresponding locations, or the compiler will not
treat it as a quantum gate but instead of the original Julia expression.

## Why?

There are a few reasons that we need a fully compiled DSL now.

### 1. Extensibility

Things in YaoBlocks like

```
function apply!(r::AbstractRegister, pb::PutBlock{N}) where {N}
    _check_size(r, pb)
    instruct!(r, mat_matchreg(r, pb.content), pb.locs)
    return r
end

# specialization
for G in [:X, :Y, :Z, :T, :S, :Sdag, :Tdag]
    GT = Expr(:(.), :ConstGate, QuoteNode(Symbol(G, :Gate)))
    @eval function apply!(r::AbstractRegister, pb::PutBlock{N,C,<:$GT}) where {N,C}
        _check_size(r, pb)
        instruct!(r, Val($(QuoteNode(G))), pb.locs)
        return r
    end
end
```

cannot be easily extended without define new dispatch on specialized instruction. Similarly, as long as there is a new instruction in low level, one need to redefine the dispatch in `YaoBlocks` however this is not necessary!
