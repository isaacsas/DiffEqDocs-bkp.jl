# Discrete Stochastic (Gillespie) Equations

In this tutorial we will describe how to define and simulate continuous-time
discrete-state jump processes, also known in biological fields as Gillespie-type
models. The underlying algorithms that sample these processes are Gillespie-type
methods, also known as stochastic simulation algorithms (SSAs). This tutorial
assumes you have read the [Ordinary Differential Equations
tutorial](ode_example.html). The discrete stochastic simulations we consider are
a form of jump equation with a "trivial" (non-existent) differential equation.
We will first demonstrate how to build these types of models using the
biological modeling functionality, then describe how to build it directly
and more generally using jumps, and finally show how to add discrete stochastic
simulations to differential equation models.

## Defining a Model using Reactions

For our example, we will build an SIR model which matches the tutorial from
[Gillespie.jl](https://github.com/sdwfrost/Gillespie.jl). SIR stands for
susceptible, infected, and recovered, and is a model of disease spread. When a
susceptible person comes in contact with an infected person, the disease has a
chance of infecting the susceptible person. This "chance" is determined by the
number of susceptible persons and the number of infected persons, since in
larger populations there is a greater chance that two people come into contact.
Normally, the rate is modeled as the amount

```julia
rate_constant*num_of_susceptible_people*num_of_infected_people
```

The `rate_constant` is determined by other factors like the type of the disease.
It can be interpreted as the probability per time one pair of susceptible and
infected people encounter each other, with the susceptible person becoming sick.
The overall rate (i.e. probability per time) that some susceptible person gets
sick is then given by the rate constant multiplied by the number of possible
pairs of susceptible and infected people. This formulation is known as the [law
of mass action](https://en.wikipedia.org/wiki/Law_of_mass_action).


Let `s` be the number of susceptible persons, `i` be the number of infected
persons, and `r` be the number of recovered persons. In this case, we can
re-write our overall rate as:

```julia
rate_constant*s*i
```

Thus we have that our "reactants" are components 1 and 2. When this "reaction"
occurs, the result is that one susceptible person turns into an infected person.
We can think of this as doing:

```julia
s -= 1
i += 1
```

that is, we decrease the number of susceptible persons by 1 and increase the number
of infected persons by 1.

These are the facts the are encoded in the reaction:

```
c1, s + i --> 2i
```

This "reaction" encodes that a susceptible person and an infected person can
interact, resulting in two infected persons (i.e. the susceptible person was
infected). Here, `c1` is the reaction constant.

To finish the model, we define one more reaction. Over time, infected people become
less infected. The chance that any one person heals during some time unit depends
on the number of people who are infected. Thus the rate at which infected people
turn into recovered people is

```julia
rate_constant*i
```

When this happens, we lose one infected person and gain a recovered person. This
reaction is thus modeled as:

```julia
c2, i --> r
```

Thus our full reaction network is:

```julia
sir_model = @reaction_network SIR begin
    c1, s + i --> 2i
    c2, i --> r
end c1 c2
```

Notice that the order the variables are introduced in the model is `s`, then
`i`, then `r`, and thus this is the canonical ordering of the variables.

## Building and Solving the Problem

First, we have to define some kind of differential equation that we can "solve"
to simulate the jump process. Since we do not want any continuous changes in the
numbers of the different types of people, we will build a `DiscreteProblem`. We
do this by giving the constructor `u0`, the initial condition, and `tspan`, the
timespan. Here, we will start with `999` susceptible people, `1` infected
person, and `0` recovered people, and solve the problem from `t=0.0` to
`t=250.0`. We use the parameters `c1 = 0.1/1000` and `c2 = 0.01`. Thus we build
the problem via:

```julia
p = (0.1/1000,0.01)
prob = DiscreteProblem([999,1,0],(0.0,250.0),p)
```

The reaction network can be converted into various differential equations
like `JumpProblem`, `ODEProblem`, or an `SDEProblem`. To turn it into a
jump problem, we simply do:

```julia
jump_prob = JumpProblem(prob,Direct(),sir_model)
```

This is now a problem that can be solved using the differential equations solvers.
Since our problem is discrete, we will use the `FunctionMap()` method.

```julia
sol = solve(jump_prob,FunctionMap())
```

This solve command takes the standard commands of the common interface, and the
solution object acts just like any other differential equation solution. Thus
there exists a plot recipe, which we can plot with:

```julia
using Plots; plot(sol)
```

![gillespie_solution](../assets/gillespie_solution.png)

## SSAStepper

The previous example used `FunctionMap()` to perform the jump process
simulation. `FunctionMap` is a `DiscreteProblem` algorithm in OrdinaryDiffEq.jl.
This shows that any common interface algorithm can be used to perform the
timestepping since this is implemented over the callback interface. In many
cases we may have a pure jump system that only involves `ConstantRateJump`s
and/or `MassActionJump`s. When that's the case, a substantial performance benefit 
may be gained by using `SSAStepper()`

```julia
sol = solve(jump_prob,SSAStepper())
```

`SSAStepper` is a barebones SSA method which doesn't allow defining events or
integrating simultaneous ODEs, but is very efficient for pure jump/SSA problems.

## Controlling Saving Behavior

Note that jumps act via the callback interface which defaults to saving at each event.
The reason is because this is required in order to accurately resolve every discontinuity
exactly (and this is what allows for perfectly vertical lines!). However, in many cases
when using jump problems you may wish to decrease the saving pressure given by large
numbers of jumps. To do this, you set `save_positions` in the `JumpProblem`. Just like
for other callbacks, this is a tuple `(bool1,bool2)` which saves whether to save
before or after a jump. If we do not want to save at every jump, we would thus pass:

```julia
jump_prob = JumpProblem(prob,Direct(),sir_model,save_positions=(false,false))
```

Now the saving controls associated with the integrator are the only ones to note.
For example, we can use `saveat=0.5` to save at an evenly spaced grid:

```julia
sol = solve(jump_prob,FunctionMap(),saveat=0.5)
```

## Defining the Jumps Directly: ConstantRateJump

Instead of using the biological modeling functionality of `@reaction_network`, we can
directly define jumps. This allows for more general types of rates, at the cost
of some modeling friendliness. The constructor for a `ConstantRateJump` is:

```julia
jump = ConstantRateJump(rate,affect!)
```

where `rate` is a function `rate(u,p,t)` and `affect!` is a function of the integrator
`affect!(integrator)` (for details on the integrator, see the
[integrator interface docs](http://docs.juliadiffeq.org/latest/basics/integrator.html)).
Thus, to define the jump equivalents to the above reactions, we can use:

```julia
rate1(u,p,t) = (0.1/1000.0)*u[1]*u[2]
function affect1!(integrator)
  integrator.u[1] -= 1
  integrator.u[2] += 1
end
jump = ConstantRateJump(rate1,affect1!)

rate2(u,p,t) = 0.01u[2]
function affect2!(integrator)
  integrator.u[2] -= 1
  integrator.u[3] += 1
end
jump2 = ConstantRateJump(rate2,affect2!)
```

We can then use `JumpProblem` to augment a problem with jumps. To add the jumps
to the `DiscreteProblem` and solve it, we would simply do:

```julia
jump_prob = JumpProblem(prob,Direct(),jump,jump2)
sol = solve(jump_prob,FunctionMap())
```
Note, in systems with more than a few jumps (more than ~10), it can be
advantageous to use a different internal representation for the jump collection.
For such systems it is recommended to use `DirectFW()`, which should offer
better performance than `Direct()`.

## Defining the Jumps Directly: MassActionJump
For systems that can be represented as mass action reactions, a further
specialization of the jump type is possible that offers improved computational
performance; `MasssActionJump`. Suppose the system has ``N`` chemical species
``\{S_1,\dots,S_N\}``. A general mass action reaction has the form

```math
R_1 S_1 + R_2 S_2 \dots R_N S_N \overset{k}{\rightarrow} P_1 S_1 + P_2 S_2 + \dots P_N S_N
```
where the non-negative integers ``(R_1,\dots,R_N\)`` denote the *reactant
stoichiometry* of the reaction, and the non-negative integers
``(P_1,\dots,\P_N)`` the *product stoichiometry*. The *net stoichiometry* is
the net change in each chemical species from the reaction occurring one time,
given by ``(P_1-R_1,\dots,P_N-R_N)``.

As an example, consider again the SIR model defined in the `@reaction_network`
above. The species are then `(s,i,r)`. The first reaction has rate `c1`,
reactant stoichiometry `(1,1,0)`, product stoichiometry `(0,2,0)`, and net
stoichiometry `(-1,1,0)`. The second reaction has rate `c2`, reactant
stoichiometry `(0,1,0)`, product stoichiometry `(0,0,1)`, and net stoichiometry
`(0,-1,1)`.


## Defining the Jumps Directly: Mixing ConstantJump and MassActionJump

## Adding Jumps to a Differential Equation

Notice that if we instead used some form of differential equation instead of a
`DiscreteProblem`, we would add the jumps/reactions to the differential equation.
Let's define an ODE problem, where the continuous part only acts on some new
4th component:

```julia
function f(du,u,p,t)
  du[4] = u[2]*u[3]/100000 - u[1]*u[2]/100000
end

prob = ODEProblem(f,[999.0,1.0,0.0,100.0],(0.0,250.0))
```

Notice we gave the 4th component a starting value of 100. The same steps as above
will thus solve this hybrid equation. For example, we can solve it using the
`Tsit5()` method via:

```julia
jump_prob = GillespieProblem(prob,Direct(),r1,r2)
sol = solve(jump_prob,Tsit5())
```

![gillespie_ode](../assets/gillespie_ode.png)

### Caution about Constant Rate Jumps

Note that the assumption which is required for constant rate jumps is that their
reaction rates must be constant on the interval between any constant rate jumps.
Thus in the examples above,

```julia
rate(u,p,t) = (0.1/1000.0)*u[1]*u[2]
rate(u,p,t) = 0.01u[2]
```

both must be constant other than changes due to some constant rate jump (the same
applies to reactions). Since these rates only change when `u[1]` or `u[2]` is changed,
and `u[1]` and `u[2]` only change when one of the jumps occur, this setup is valid.
However, `t*(0.1/1000.0)*u[1]*u[2]` would not be valid because the rate would change
during the interval, as would `(0.1/1000.0)*u[1]*u[4]`. Thus one must be careful
about to follow this rule when choosing rates.

(but note that it's okay for `u[4]` to depend on the other variables because its
updated in a continuous manner!)

If your problem must have the rates depend on a continuously changing quantity,
you need to use the `VariableRateJump` or `VariableRateReaction` instead.

## Adding a VariableRateJump

Now let's consider adding a reaction whose rate changes continuously with the
differential equation. To continue our example, let's let there be a new reaction
which has the same effect as `r2`, but now is dependent on the amount of `u[4]`.

```julia
rate3(u,p,t) = 1e-2u[4]
function affect3!(integrator)
  integrator.u[2] -= 1
  integrator.u[3] += 1
end
jump3 = VariableRateJump(rate3,affect3!)
```

We would expect this reaction to increase the amount of transitions from state
2 to 3. Solving the equation is exactly the same:

```julia
prob = ODEProblem(f,[999.0,1.0,0.0,1.0],(0.0,250.0))
jump_prob = JumpProblem(prob,Direct(),jump,jump2,jump3)
sol = solve(jump_prob,Tsit5())
```

![variable_rate_gillespie](../assets/variable_rate_gillespie.png)

Notice that this increases the amount of 3 at the end, reducing the falloff in
the rate (though this model is kind of nonsensical).

Note that even if the problem is a `DiscreteProblem`, `VariableRateJump`s and
`VariableRateReaction`s require a continuous solver, like an ODE/SDE/DDE/DAE solver.

Lastly, we are not restricted to ODEs. For example, we can solve the same jump
problem except with multiplicative noise on `u[4]` by using an `SDEProblem` instead:

```julia
function g(du,u,p,t)
  du[4] = 0.1u[4]
end

prob = SDEProblem(f,g,[999.0,1.0,0.0,1.0],(0.0,250.0))
jump_prob = JumpProblem(prob,Direct(),jump,jump2,jump3)
sol = solve(jump_prob,SRIW1())
```

![sde_gillespie](../assets/sde_gillespie.png)

## RegularJumps and Tau-Leaping

The previous parts described how to use `ConstantRateJump` and `VariableRateJump`
to add jumps to differential equation algorithms over the callback interface.
However, in many cases you do not need to step to every jump time. Instead,
regular jumping allows you to pool together jumps and perform larger updates
in a statistically-correct but more efficient manner.

For `RegularJump`s, we pool together the jumps we wish to perform. Here our
`rate` is a vector equation which computes the rates of each jump process
together:

```julia
function rate(out,u,p,t)
    out[1] = (0.1/1000.0)*u[1]*u[2]
    out[2] = 0.01u[2]
end
```

and then we compute the total change matrix `c`

```julia
function c(dc,u,p,t,mark)
    dc[1,1] = -1
    dc[2,1] = 1
    dc[2,2] = -1
    dc[3,2] = 1
end
```

where each column is a different jump process. We then declare the form of `dc`
and build a `RegularJump`:

```julia
dc = zeros(3,2)
rj = RegularJump(rate,c,dc;constant_c=true)
```

From there we build a `JumpProblem`:

```julia
prob = DiscreteProblem([999.0,1.0,0.0],(0.0,250.0))
jump_prob = JumpProblem(prob,Direct(),rj)
```

Note that when a `JumpProblem` has a `RegularJump`, special algorithms are
required. This is detailed on
[the jump solvers page](http://docs.juliadiffeq.org/latest/solvers/jump_solve.html).
One such algorithm is `SimpleTauLeaping`, which we use as follows:

```julia
sol = solve(jump_prob,SimpleTauLeaping();dt=1.0)
```
