---
layout: post
title:  FluxDiffEq.jl: A Julia Library for Neural Differential Equations
author: Chris Rackauckas, Mike Innes, Yingbo Ma, Lyndon White, Vaibhav Dixit
---

The [Neural Ordinary Differential Equations](https://arxiv.org/abs/1806.07366)
paper has been a hot topic even before it made a splash as Best Paper of
NeurIPS 2018. By mixing ordinary differential equations and neural networks
they were able to improve training speeds and accuracy over ResNet. Now with the
floodgates opened causing a merge of differential equations merging with machine
learning, the purpose of this blog post is into introduce the reader to
differential equations from a data science perspective and show how to mix these
tools with neural nets.

The advantages of the Julia [DifferentialEquations.jl](https://github.com/JuliaDiffEq/DifferentialEquations.jl) library for numerically solving
differential equations have been
[discussed in detail in other posts](http://www.stochasticlifestyle.com/comparison-differential-equation-solver-suites-matlab-r-julia-python-c-fortran/).
Recently, these native Julia differential equation solvers have successfully been embedded
into the [Flux.jl](https://github.com/FluxML/Flux.jl/) deep learning package, to allow the use of a full suite of
highly tested and optimized DiffEq methods within neural networks. Using the new package
[DiffEqFlux.jl](https://github.com/JuliaDiffEq/DiffEqFlux.jl/), we will show the reader how to easily add differential equation
layers to neural networks using a range of differential equations models, including stiff ordinary
differential equations, stochastic differential equations, delay differential
equations, and hybrid (discontinuous) differential equations.

This is the first user-friendly toolbox to combine state-of-the-art differential
equations and neural networks seamlessly together. The blog post will also show
why the flexibility of a full differential equation solver suite is necessary
for doing this properly. With the ability to solve fuse neural networks
seemlessly with ODEs, SDEs, DAEs, DDEs, stiff equations, and different methods
for adjoint sensitivity calculations, this is a large generalization of the
neural ODEs work and will allow researchers to better explore the problem
domain.

(Note: If you are interested in this work and are an undergraduate or graduate
student, we have Google Summer of Code projects available in this area. This
[pays quite handsomely over the summer](https://developers.google.com/open-source/gsoc/help/student-stipends).
Please join the [Julia Slack](https://slackinvite.julialang.org/) and the #jsoc channel to discuss in more detail.)

## Why differential equations and how are they related to machine learning models?

The first question someone not familiar with the field might ask is, why are
differential equations important in this context? The simple answer is that a
differential equation is a way to specify an arbitrary nonlinear transform via
its structure. Let's unpack that statement a bit. There are generally three ways
to define a nonlinear transformation. The first is to explicitly say what it is.
For example, the sigmoid function is $\sigma(x)=\frac{e^x}{e^x + 1}$.This only works
if you know the exact functional form that relates the input to the output.
However, in many cases, such exact relations are not known a priori.
So how do you do nonlinear modeling if you don't know the nonlinearity?

There are two separate ways to then handle this problem. One method is machine
learning. Usually this is presented as you have some $x$ and you want to predict
$y$. This generation of a prediction from $x$ is the machine learning model, call it
$ML$, where you put $x$ into the program and it spits out values $y$. But if you think
about it differently, this is actually just a nonlinear transformation $y=ML(x)$.
The reason why the model $ML$ is interesting is because its form is basic but adapts to the
data itself. For example, a simple neural network (in design matrix form) with
sigmoid activation functions then the model is simply matrix multiplications followed
by sigmoids, meaning $ML(x)=\sigma(W_{3}\cdot\sigma(W_{2}\cdot\sigma(W_{1}\cdot x)))$ is a three-layer deep
neural network, where $W=(W_1,W_2,W_3)$ are learnable parameters. You then choose $W$ such that
$ML(x)=y$ reasonably fits the function you wanted it to fit. The theory and practice of
machine learning confirms that this is a good way to learn nonlinearities.
For example, the Universal Approximation Theorem states that, for
enough layers or enough parameters (i.e. sufficiently large $W_{i}$ matrices), $ML(x)$
can approximate any nonlinear function sufficiently close (subject to common constraints).

So great, this always works! But it has some caveats, the main being
that it has to learn everything about the nonlinear transform directly from the
data. In many cases we might not know the full nonlinear equation, but we may
know details about its structure. For example, the nonlinear function could be
the population of rabbits in the forest, and we might know that their rate of births
is dependent on the current population. Thus instead of starting from nothing,
we may want to use this known a priori relation and a set of parameters that defines it. For the
rabbits, let's say that we want to learn

$$\text{rabbits tomorrow} = \text{Model}(\text{rabbits today}).$$

In this case, we have prior knowledge of the rate of births being dependent on
the current population. The way to mathematically state this
structural assumption is via a differential equation. Here, what we are saying
is that the birth rate of the rabbit population at a given time point increases
when we have more rabbits. The simplest way of encoding that is

$$\text{rabbits}'(t) = \alpha\cdot \text{rabbits}(t)$$

where $\alpha$ is some learnable constant. If you know your calculus, the solution
here is exponential growth from the starting point with a growth rate $\alpha$:
$\text{rabbits}(t_\text{start})e^(\alpha t)$. But notice that we didn't need to know the
solution to the differential equation to validate the idea: we encoded the
structure of the model and mathematics itself then outputs what the solution
should be. Because of this, differential equations have been the tool of choice
in most science. For example, physical laws tell you how electrical quantities
emit forces ([Maxwell's Equations](https://en.wikipedia.org/wiki/Maxwell%27s_equations)).
These are essentially equations of how things change and thus
"where things will be" is the solution to a differential equation. But in recent
decades this application has gone much further, with fields like systems biology
learning about cellular interactions by encoding known biological structures and
mathematically enumerating our assumptions or in targeted drug dosage through
PK/PD modelling in systems pharmacology.

So as our machine learning models grow and are hungry for larger and larger
amounts of data, differential equations have become an attractive option for
specifying nonlinearities in a learnable (via the parameters) but constrained
form. They are essentially a way of incorporating prior domain-specific knowledge of the structural relations
between the inputs and outputs. Given this way of looking at the two, both methods
tradeoff advantages and disadvantages, making them complementary tools for modeling.
It seems like a clear next step in scientific practice to start putting them
together in new and exciting ways!

## What is the Neural Ordinary Differential Equation (ODE)?

The neural ordinary differential equation is one of many ways to put these two
subjects together. The simplest way of explaining it is that, instead of
learning the nonlinear transformation directly, we wish to learn the structures
of the nonlinear transformation. Thus instead of doing $y=ML(x)$, we put the
machine learning model on the derivative, $y'(x) = ML(x)$, and now solve the ODE.
Why would you ever do this? Well, one motivation is that, if you were to do this
and then solve the ODE using the simplest and most error prone method, the
Euler method, what you get is equivalent to a recurrent neural network. The way
the Euler method works is to notice that $y'(x) = \frac{dy}{dx}$, then write

$$\Delta y = (y_\text{next} - y_\text{prev}) = \Delta x\cdot ML(x) \text{ which implies that } y_{i+1} = y_{i} + \Delta x\cdot ML(x_{i}).$$

Looking familiar now? This is the basis of an RNN, the foundation of ResNet and
all of the best AI systems known to date. The genius of the neural ordinary
differential equation paper was to notice this and say, let's just model the
differential equation directly and then solve it using a much better numerical
ODE solver method. The creation of numerical ODE solvers is a science that goes
all the way back to the first computers, and can adaptively choose step sizes $\Delta x$
and use high order approximations to reduce the number of steps required to get
sufficiently small error. By swapping out the Euler discretization with the LSODE
numerical ODE solver, their results were good enough to make everyone think twice
about reflexively using an RNN.

## How do you solve an ODE?

First, how do you numerically specify and solve an ODE? If you're new to solving
ODEs, you may want to watch our
[video tutorial on solving ODEs in Julia](https://www.youtube.com/watch?v=KPEqYtEd-zY)
and look through the
[ODE tutorial of the DifferentialEquations.jl documentation](http://docs.juliadiffeq.org/latest/tutorials/ode_example.html).
The idea is that you define an `ODEProblem` via a derivative equation `u'=f(u,p,t)`,
and provide an initial condition `u0`, and a timespan `tspan` to solve over , and
specify the parameters `p`.

For example, the
[Lotka-Volerra equations describe the dynamics of the population of  rabbits and wolves](https://en.wikipedia.org/wiki/Lotka%E2%80%93Volterra_equations).
They can be written as:

$$
Put Lotka-Volterra equations
$$

and encoded in Julia like:

```julia
using DifferentialEquations
function lotka_volterra(du,u,p,t)
  x, y = u
  α, β, δ, γ = p
  du[1] = dx = α*x - β*x*y
  du[2] = dy = -δ*y + γ*x*y
end
u0 = [1.0,1.0]
tspan = (0.0,10.0)
p = [1.5,1.0,3.0,1.0]
prob = ODEProblem(lotka_volterra,u0,tspan,p)
```

Then to solve the differential equations, you can simply call `solve` on the
`prob`:

```julia
sol = solve(prob)
using Plots
plot(sol)
```

![LV Solution Plot](https://user-images.githubusercontent.com/1814174/51388169-9a07f300-1af6-11e9-8c6c-83c41e81d11c.png)

Optional arguments can be passed to control the ODE solver to customise most aspects of the numerical integration.
For example, if you've some familiarity with the area, a good choice for
this equation is to use the 5th order Runge-Kutta method `Tsit5()` (related to the `dopri` method,
or `ode45` in MATLAB). We can also say that we want this solved with a relative error
tolerance of `1e-8`, and that we want the solution saved at every `0.1` time points. This looks like:

```julia
sol = solve(prob,Tsit5(),saveat=0.1,reltol=1e-8)
```

There are a lot more arguments you can pass in to customise most aspects of the ODE solver,
as
[specified in the documentation](http://docs.juliadiffeq.org/latest/basics/common_solver_opts.html),
but these will suffice for our demonstration.

Note that the solution of a differential equation is a continuous object, so
for example

```julia
sol(5.24)
```

gives the solution at time `5.24`. It also acts as an array, where

```julia
sol[2,5]
sol.t[5]
```

is the solution of the 2nd variable (`y`) at the the 5th time point. Notice that
the 5th time point is `0.4` since we set `saveat=0.1`. If we did not set `saveat`,
this would be adaptively chosen. More details on handling the solution type
[can be found in the DifferentialEquations.jl documentation](http://docs.juliadiffeq.org/latest/basics/solution.html).

One last thing to note is that we can make our initial condition (`u0`) and time spans (`tspans`)
to be functions of the parameters (the elements of `p`). For example, we can define the `ODEProblem`:

```julia
u0_f(p,t0) = [p[2],p[4]]
tspan_f(p) = (0.0,10*p[4])
p = [1.5,1.0,3.0,1.0]
prob = ODEProblem(lotka_volterra,u0_f,tspan_f,p)
```

In this form, everything about the problem is determined by the parameter vector (`p`, referred to
as `θ` in associated literature). The utility of this will be seen later.

## Let's Put an ODE Into a Neural Net Framework!

To understand embedding an ODE into a neural network, let's recall what a layer
of a neural network actually is. A layer is a transformation 𝐑ⁿ→𝐑ᵐ which
takes in a vector of size `n` and spits out a new vector of size `m`. That's it!
So a differential equation layer is simply a layer that takes in a parameter
vector `p` (of size `n`) and spits out a new vector (of size `m`). You can do this
manually using the
[`remake` function in DifferentialEquations.jl](http://docs.juliadiffeq.org/latest/basics/problem.html#Modification-of-problem-types-1),
or you can use the Flux.jl layers defined in DiffEqFlux.jl. Let's give those a spin.

The most basic differential equation layer is `diffeq_rd`. The function is of
the form `diffeq_rd(p,reduction,prob,args...;kwargs...)`. The way to understand this
function is that it does the following. It takes in the parameter vector `p`,
it puts it in the differential equation defined by `prob`, solves it with
the chosen arguments (solver, tolerance, etc), and then gets a vector out through
some function `reduction`. For example, in our Lotka-Volerra problem, let's say we wanted
to go from `p` to a vector of an evenly-spaced time series for `x(t)`.
In the REPL, we would write:

```julia
p = [1.5,1.0,3.0,1.0]
prob = ODEProblem(lotka_volterra,u0,tspan,p)
sol = solve(prob,Tsit5(),saveat=0.1)
A = sol[1,:] # length 101 vector
```

Let's plot `(t,A)` over the ODE's solution to see what we got:

```julia
plot(sol)
t = 0:0.1:10.0
scatter!(t,A)
```

![Data points plot](https://user-images.githubusercontent.com/1814174/51388173-9c6a4d00-1af6-11e9-9878-3c585d3cfffe.png)

So this is what we wanted: a vector which is the time series of `x(t)` saved at every 0.1 time steps
given that the parameters are `p`. To write this using the layer function, we simply make
our solution handling be in the function `reduction` used above in `diffeq_rd(p,reduction,prob,args...;kwargs...)`.
Thus this code is equivalent to:

```julia
using FluxDiffEq
reduction(sol) = sol[1,:]
diffeq_rd(p,reduction,prob,Tsit5(),saveat=0.1)
```

As a reminder, this is a function which takes in a parameter vector `p`,
solves the Lotka-Volterra problem `prob` with our chosen ODE solver arguments,
and then gets a vector out via `reduction` which is the time series of `x(t)`.

The nice thing about `diffeq_rd` is that it takes care of the type handling
necessary to make it compatible with the neural network framework (here Flux.jl). To show this,
let's define a neural network with the function as our single layer, and then a loss
function that is the squared distance of the output values from `1`. In Flux, this looks like:

```julia
p = param([2.2, 1.0, 2.0, 0.4]) # Initial Parameter Vector
params = Flux.Params([p])

function predict_rd() # Our 1-layer neural network
  diffeq_rd(p,reduction,prob,Tsit5(),saveat=0.1)
end

loss_rd() = sum(abs2,x-1 for x in predict_rd()) # loss function
```

Now we tell Flux to train the neural network by running a 100 epoch
to minimise our loss function (`loss_rd()`) and thus obtain the optimized parameters:

```julia
data = Iterators.repeated((), 100)
opt = ADAM(0.1)
cb = function () #callback function to observe training
  display(loss_rd())
  # using `remake` to re-create our `prob` with current parameters `p`
  display(plot(solve(remake(prob,p=Flux.data(p)),Tsit5(),saveat=0.1),ylim=(0,6)))
end

# Display the ODE with the initial parameter values.
cb()

Flux.train!(loss_rd, params, data, opt, cb = cb)
```

![Flux ODE Training Animation](https://user-images.githubusercontent.com/1814174/51399500-1f4dd080-1b14-11e9-8c9d-144f93b6eac2.gif)

Flux.jl then finds the parameters of the neural network (`p`) which minimize
the cost function, i.e. it trains the neural network: it just so happens that here training the neural network happens to include solving an ODE.
Since our cost function put a penalty whenever the number of
rabbits was far from 1, our neural network found parameters where our population
of rabbits and wolves are both constant 1.

Now that we have solving ODEs as just a layer, we can add it anywhere. For example,
the multilayer perceptron is written in Flux.jl as

```julia
m = Chain(
  Dense(28^2, 32, relu),
  Dense(32, 10),
  softmax)
```

and if we had an appropriate ODE which took a parameter vector of the right size,
we can stick it right in there:

```julia
m = Chain(
  Dense(28^2, 32, relu),
  # this would require an ODE of 32 parameters
  p -> diffeq_rd(p,reduction,prob,Tsit5(),saveat=0.1),
  Dense(32, 10),
  softmax)
```

or we can stick it into a convolutional neural network:

```julia
m = Chain(
  Conv((2,2), 1=>16, relu),
  x -> maxpool(x, (2,2)),
  Conv((2,2), 16=>8, relu),
  x -> maxpool(x, (2,2)),
  x -> reshape(x, :, size(x, 4)),
  p -> diffeq_rd(p,reduction,prob,Tsit5(),saveat=0.1),
  Dense(288, 10), softmax) |> gpu
```

The world is your oyster.

## Why is a full ODE solver suite necessary for doing this well?

With Julia we have solved the technical issue of embedding native
differential equation solvers inside of the neural network framework. Details
of how it was done will follow, but we have to address the necessary question
first: why was this important to do at all? Or another way to put it is, why
not just re-implement a few simple ODE solvers into a neural network framework
and be done with it?

This is the strategy which was taken with
[torchdiffeq](https://github.com/rtqichen/torchdiffeq) which implements an adaptive
Runge Kutta 4-5 (`dopri5`) and an Adams-Bashforth-Moulton method (`adams`)
The issue are not sufficient for all, or even most, ODEs. A classic example of this is the
[ROBER ODE](https://www.radford.edu/~thompson/vodef90web/problems/demosnodislin/Single/DemoRobertson/demorobertson.pdf).
The most well-tested (and optimized) implementation of an Adams-Bashforth-Moulton
method is the [CVODE integrator in the C++ package SUNDIALS](https://computation.llnl.gov/projects/sundials)
(a derivative of the classic LSODE).
Let's use DifferentialEquations.jl to call CVODE with its Adams method and have
it solve the ODE for us:

```julia
rober = @ode_def Rober begin
  dy₁ = -k₁*y₁+k₃*y₂*y₃
  dy₂ =  k₁*y₁-k₂*y₂^2-k₃*y₂*y₃
  dy₃ =  k₂*y₂^2
end k₁ k₂ k₃
prob = ODEProblem(rober,[1.0;0.0;0.0],(0.0,1e11),(0.04,3e7,1e4))
solve(prob,CVODE_Adams())
```

(For those familar with solving ode's in  MATLAB, this is similar to `ode113`)

If you try the `dopri` method from
[Ernst Hairer's Fortran Suite](https://www.unige.ch/~hairer/software.html), you
get the same issue:

```julia
using ODEInterfaceDiffEq
solve(prob,dopri5())
```

(note: this is like MATLAB's `ode45`)

Both of these stall and fail to solve the equation. It's not an implementation
issue either. Our native Julia versions of these methods (`DP5()` and `VCABM`)
also exhibit the same behavior. It's actually a well-known property of these
methods on this kind of ODE.
This ODE is known to be a [stiff ODE](https://en.wikipedia.org/wiki/Stiff_equation),
and thus methods with "smaller stability regions" will not be able to solve it
appropriately (for more details, I suggest reading Hairer's Solving Ordinary
Differential Equations II).

I chose these two methods because these are the ones implemented in torchdiffeq
and mentioned in the neural ordinary differential equations paper. They are not
bad methods by any means, they are classics because of how efficient they are
on a good number of problems. But it's clear from this example that they are
not applicable to all ODEs. Instead, you need the flexibility to choose other
solvers which have different properties such as increased stiffness. For example,
when applying `KenCarp4()` to this problem, the equation is solved in a blink of
an eye:

```julia
sol = solve(prob,KenCarp4())
using Plots
plot(sol,xscale=:log10,tspan=(0.1,1e11))
```

![ROBER Plot](https://user-images.githubusercontent.com/1814174/51388944-eb18e680-1af8-11e9-874f-09478759596e.png)

Stabilizing explicit methods via PI-adaptive controllers, step prediction in
implicit solvers, etc. are all intricate details that take a lot of time and
testing in order to build methods which are efficient for the more difficult
equations. This is why very few production-quality solvers have ever been
created. For example, R's deSolve and Python's SciPy simply wrap the older
Fortran methods, as detailed
[in the ODE solver summary post](http://www.stochasticlifestyle.com/comparison-differential-equation-solver-suites-matlab-r-julia-python-c-fortran/))
Not only that, but different problems require different methods.
[Symplectic integrators](http://docs.juliadiffeq.org/latest/solvers/dynamical_solve.html#Symplectic-Integrators-1)
are required to [adequately handle physical many problems without drift](https://scicomp.stackexchange.com/questions/29149/what-does-symplectic-mean-in-reference-to-numerical-integrators-and-does-scip/29154#29154),
and tools like [IMEX integrators](http://docs.juliadiffeq.org/latest/solvers/split_ode_solve.html#Implicit-Explicit-(IMEX)-ODE-1)
are required to handle ODEs which [come from partial differential equations](https://www.youtube.com/watch?v=okGybBmihOE).
Thus building and maintaining a full set of optimized differential equation
solvers is itself an entire research programme. Given the intricacy of such
a topic, reimplementing the solvers with all of the details into every
machine learning framework is an infeasible task. By embedding the work of our
DifferentialEquations.jl project into Flux.jl, we can continue to develop the
differential equation solvers for traditional scientific computing
and the users of Flux.jl will benefit from all of our work.

## What kinds of differential equations are there?

Ordinary differential equations are only one kind of differential equation. There
are many additional features you can add to the structure of a differential equation.
For example, the amount of bunnies in the future isn't dependent on the number
of bunnies right now because it takes a non-zero amount of time for a parent
to come to term after a child is incepted. Thus the birth rate of bunnies is
actually due to the amount of bunnies in the past. Using a lag term in a
differential equation's derivative makes this equation known as a delay
differential equation (DDE). Since
[DifferentialEquations.jl handles DDEs](http://docs.juliadiffeq.org/latest/tutorials/dde_example.html)
through the same interface as ODEs, it they can be used as a layer in
Flux.jl as well. Here's an example:

```julia
function delay_lotka_volterra(du,u,h,p,t)
  x, y = u
  α, β, δ, γ = p
  du[1] = dx = (α - β*y)*h(p,t-0.1)[1]
  du[2] = dy = (δ*x - γ)*y
end
h(p,t) = ones(eltype(p),2)
prob = DDEProblem(delay_lotka_volterra,[1.0,1.0],h,(0.0,10.0),constant_lags=[0.1])

p = param([2.2, 1.0, 2.0, 0.4])
params = Flux.Params([p])
function predict_rd_dde()
  diffeq_fd(p,reduction,prob,101,MethodOfSteps(Tsit5()),saveat=0.1)
end
loss_fd_dde() = sum(abs2,x-1 for x in predict_fd_dde())
loss_fd_dde()
```

Additionally we can add randomness to our differential equation to simulate
how random events can cause extra births or more deaths than expected. This
kind of equation is known as a stochastic differential equation (SDE).
Since [DifferentialEquations.jl handles SDEs](http://docs.juliadiffeq.org/latest/tutorials/sde_example.html)
(and is currently the only library with adaptive stiff and non-stiff SDE integrators),
these can be handled as a layer in Flux.jl similarly. Here's a neural net layer
with an SDE:

```julia
function lotka_volterra_noise(du,u,p,t)
  du[1] = 0.1u[1]
  du[2] = 0.1u[2]
end
prob = SDEProblem(lotka_volterra,lotka_volterra_noise,[1.0,1.0],(0.0,10.0))

p = param([2.2, 1.0, 2.0, 0.4])
params = Flux.Params([p])
function predict_fd_sde()
  diffeq_fd(p,reduction,101,prob,SOSRI(),saveat=0.1)
end
loss_fd_sde() = sum(abs2,x-1 for x in predict_fd_sde())
loss_fd_sde()
```

And we can train the neural net to watch it in action and find parameters to make
the amount of bunnies close to constant:

```julia
data = Iterators.repeated((), 100)
opt = ADAM(0.1)
cb = function ()
  display(loss_fd_sde())
  display(plot(solve(remake(prob,p=Flux.data(p)),SOSRI(),saveat=0.1),ylim=(0,6)))
end

# Display the ODE with the current parameter values.
cb()

Flux.train!(loss_fd_sde, params, data, opt, cb = cb)
```

![SDE NN Animation](https://user-images.githubusercontent.com/1814174/51399524-2c6abf80-1b14-11e9-96ae-0192f7debd03.gif)

And we can keep going. There are differential equations
[which are piecewise constant](http://docs.juliadiffeq.org/latest/tutorials/discrete_stochastic_example.html)
used in biological simulations, or
[jump diffusion equations from financial models](http://docs.juliadiffeq.org/latest/tutorials/jump_diffusion.html),
and the solvers map right over to the Flux.jl neural network frame work through FluxDiffEq.jl
It is worth noting that FluxDiffEq.jl uses is only around ~50 lines of code to pull this all off.

## Implementing the Neural ODE layer in Julia

Let's go all the way back for a second and now implement the neural ODE layer
in Julia. Remember that this is simply an ODE where the derivative function
is defined by a neural network itself. To do this, let's first define the
neural net for the derivative. We can use a multilayer ...

## The core technical challenge: backpropogation through differential equation solvers

Let's end by explaining the technical issue that needed a solution to make this
all possible. The core to any neural network framework is the ability to
backpropogate derivatives in order to calculate the gradient of the loss function
with respect to the network's parameters. Thus if we stick an ODE solver as a
layer in a neural network, we need to backpropogate through it.

There are multiple ways to do this. The most common is known as (adjoint) sensitivity
analysis. Sensitivity analysis defines a new ODE whose solution gives the
gradients to the cost function w.r.t. the parameters, and solves this secondary
ODE. This is the method discussed in the neural ordinary differential equations
paper, but actually dates back much further, and popular ODE solver frameworks
like [FATODE](http://people.cs.vt.edu/~asandu/Software/FATODE/index.html),
[CASADI](https://web.casadi.org/), and
[CVODES](https://computation.llnl.gov/projects/sundials/cvodes)
have been available with this adjoint method for a long time (CVODES came out
in 2005!). [DifferentialEquations.jl has sensitivity analysis implemented too](http://docs.juliadiffeq.org/latest/analysis/sensitivity.html)

The problem with adjoint sensitivity analysis methods is that they require
multiple forward solutions of the ODE. As you would expect, this is very costly.
Methods like the checkpointing scheme in CVODES reduce the cost by saving closer
time points to make the forward solutions shorter at the cost of using more
memory. The method in the neural ordinary differential equations paper tries to
eliminate the need for these forward solutions by doing a backwards solution
of the ODE itself along with the adjoints. The issue with this is that this
method implicitly makes the assumption that the ODE integrator is
[reversible](https://www.physics.drexel.edu/~valliere/PHYS305/Diff_Eq_Integrators/time_reversal/).
Sadly, there are no reversible adaptive integrators for first-order ODEs, so
with no ODE solver method is this guaranteed to work. For example, here's a quick
equation where a backwards solution to the ODE using the Adams method from the
paper has >1700% error in its final point, even with solver tolerances of 1e-12:

```julia
using Sundials, DiffEqBase
function lorenz(du,u,p,t)
 du[1] = 10.0*(u[2]-u[1])
 du[2] = u[1]*(28.0-u[3]) - u[2]
 du[3] = u[1]*u[2] - (8/3)*u[3]
end
u0 = [1.0;0.0;0.0]
tspan = (0.0,100.0)
prob = ODEProblem(lorenz,u0,tspan)
sol = solve(prob,CVODE_Adams(),reltol=1e-12,abstol=1e-12)
prob2 = ODEProblem(lorenz,sol[end],(100.0,0.0))
sol = solve(prob,CVODE_Adams(),reltol=1e-12,abstol=1e-12)
@show sol[end]-u0 #[-17.5445, -14.7706, 39.7985]
```

(here we once again use the CVODE C++ solvers from SUNDIALS since they are a close
match to the SciPy integrators used in the neural ODE paper)

This inaccuracy is the reason why the method from the neural ODE paper is not
implemented in software suites, but it once again highlights a detail. Not
all ODEs will have a large error due to this issue. And on ODEs where it's not
a problem, this will be the most efficient way to do adjoint sensitivity
analysis. Thus once again we arive at the conclusion that one method is not
enough.

In DifferentialEquations.jl we are investigating many different methods for
computing the derivatives of differential equations with respect to parameters.
We have a [recent preprint](https://arxiv.org/abs/1812.01892) detailing
some of these results. One of the things we have found is that direct use of
automatic differentiation can be one of the most efficient and flexible methods.
Julia's ForwardDiff.jl, Flux.jl, and ReverseDiff.jl can directly be applied to
perform automatic differentiation on the native Julia differential equation
solvers themselves, and this can increase performance while giving new features.
Our findings show that forward-mode automatic differentiation is fastest when
there are less than 100 parameters in the differential equations, and that for
large enough number of parameters adjoint sensitivity analysis is the most efficient. Even
then, we have good reason to believe that
[the next generation reverse-mode automatic differentiation via source-to-source AD, Zygote.jl](https://julialang.org/blog/2018/12/ml-language-compiler),
will be more efficient than all of the adjoint sensitivity implementations for
large numbers of parameters.

Thus being able to switch between different gradient methods without changing
the rest of your code is crucial for having a scalable, optimized, and
maintainable framework for integrating differential equations and neural networks.
And this is precisely what FluxDiffEq.jl is doing. There are three functions:

- `diffeq_rd` uses Flux.jl's reverse-mode AD through the differential equation
  solver.
- `diffeq_fd` uses ForwardDiff.jl's forward-mode AD through the differential
  equation solver.
- `diffeq_adjoint` uses adjoint sensitivity analysis to "backprop the ODE solver"

With a similar API. Thus to switch from a reverse-mode AD layer to a forward-mode
AD layer, one simply has to change a single character. Since Julia-based automatic
differentiation works on Julia code, the native Julia differential equation
solvers will continue to benefit from advances in this field.

## Conclusion

Machine learning and differential equations are destined to come together due to
their complementary ways of describing a nonlinear world. In the Julia ecosystem
we have merged the differential equation and deep learning packages in such a
way that new developments in the two domains can directly be used together.
We are only beginning to understand the possibilities that have opened up with
this software. We hope that future blog posts will detail some of the cool
applications which mix the two disciplines, such as embedding our coming
pharmacometric simulation engine [PuMaS.jl](https://doi.org/10.1007/s10928-018-9606-9)
into the deep learning framework. With access to the full range of solvers for ODEs,
SDEs, DAEs, DDEs, PDEs, discrete stochastic equations, and more, we are
interested to see what kinds of next generation neural networks you build with Julia.
