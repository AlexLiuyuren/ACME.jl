# User Guide

## Element Creation

All circuit elements are created by calling corresponding functions; see the
[Element Reference](@ref) for details.

## Circuit Description

Circuits are described using `Circuit` instances, which are most easily created
using the `@circuit` macro:
```@docs
@circuit
```
The pins provided by each type of element are described in the [Element Reference](@ref).

Instead of or in addition to using the `@circuit` macro, `Circuit` instances can
also be populated and modified programmatically using the following functions:
```@docs
add!
delete!
connect!
disconnect!
```

For example, a cascade of 20 RC-lowpasses could be generated by:
```julia
circ = @circuit begin
    src = voltagesource(), [-] ⟷ gnd
    output = voltageprobe(), [-] ⟷ gnd
end
pin = (:src, +)
for i in 1:20
    resrefdes = add!(circ, resistor(1000))
    caprefdes = add!(circ, capacitor(10e-9))
    connect!(circ, (resrefdes, 1), pin)
    connect!(circ, (resrefdes, 2), (caprefdes, 1))
    connect!(circ, (caprefdes, 2), :gnd)
    pin = (resrefdes, 2)
end
connect!(circ, pin, (:output, +))
```

## Model Creation and Use

A `Circuit` only stores elements and information about their connections. To
simulate a circuit, a model has to be derived from it. This can be as simple
as:

```Julia
model = DiscreteModel(circ, 1/44100)
```

Here, `1/44100` denotes the sampling interval, i.e. the reciprocal of the
sampling rate at which the model should run. Optionally, one can specify the
solver to use for solving the model's non-linear equation:

```Julia
model = DiscreteModel(circ, 1/44100, HomotopySolver{SimpleSolver})
```

See [Solvers](@ref) for more information about the available solvers.

Once a model is created, it can be run:

```Julia
y = run!(model, u)
```

The input `u` is matrix with one row for each of the circuit's inputs and one
column for each time step to simulate. Likewise, the output `y` will be a
matrix with one row for each of the circuit's outputs and one column for each
simulated time step. The order of the rows will correspond to the order in which
the respective input and output elements were added to the `Circuit`. To
simulate a circuit without inputs, a matrix with zero rows may be passed:

```Julia
y = run!(model, zeros(0, 100))
```

The internal state of the model (e.g. capacitor charges) is preserved accross
calls to `run!`.

Each invocation of `run!` in this way has to allocate some memory as temporary
storage. To avoid this overhead when running the same model for many small input
blocks, a `ModelRunner` instance can be created explicitly:

```Julia
runner = ModelRunner(model, false)
run!(runner, y, u)
```

By using a pre-allocated output `y` as in the example, allocations in `run!` are
reduced to a minimum.

Upon creation of a `DiscreteModel`, its internal states (e.g. capacitor charges)
are set to zero. It is also possible to set the states to a steady state (if
one can be found) with:

```Julia
steadystate!(model)
```

This is often desirable for circuits where bias voltages are only slowly
obtained after turning them on.

## Solvers

```@docs
SimpleSolver
HomotopySolver
CachingSolver
```

The default solver used is a `HomotopySolver{CachingSolver{SimpleSolver}}`.
