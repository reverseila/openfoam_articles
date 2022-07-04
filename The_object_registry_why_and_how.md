---
author: Gerhard Holzinger
title: answer to a question on cfd-online
---

# [Why and how?](https://www.cfd-online.com/Forums/openfoam-programming-development/236671-what-object-registry-openfoam.html#post805770)

## What is the object registry?

This is quite a tough one, since it is hard to give a general answer with only
a few sentences. The questions on how and why are much easier answered than the
question of what.


## Why do we use it?

Because it is very, very conventient and it saves a lot of effort.

E.g. most solvers read and create the velocity field somewhere in the file
`createFields.H`. This causes OpenFOAM to actually read the velocity data from
disk. Later when the solver creates the turbulence model or the momentum
equation it can directly use the velocity field or pass it as a reference.


```c++
Info<< "Reading field U\n" << endl;
volVectorField U
(
    IOobject
    (
        "U",
        runTime.timeName(),
        mesh,
        IOobject::MUST_READ,
        IOobject::AUTO_WRITE
    ),
    mesh
);
```

However, imagine the solver - at run-time - uses a `functionObject` computing
the Mach number. How is the `functionObject` supposed to know about the
velocity field? Here, the object registry comes into play, as the
`functionObject` can ask the `objectRegistry` "Is there a vector field named
`U`?", if there is the `objectRegistry` returns a reference to the velocity
field.


## How it is used?

Below is some code of the `MachNo` function object. Here we see what I
discussed above: the `functionObject` asks the registry nicely if there is a
vector field of a specified name. If there is one, the `functionObject` asks
the registry for a reference to that vector field, so that it can then compute
the Mach number.


```c++
bool Foam::functionObjects::MachNo::calc()
{
    if
    (
        foundObject<volVectorField>(fieldName_)
     && foundObject<fluidThermo>(fluidThermo::dictName)
    )
    {
        const fluidThermo& thermo =
            lookupObject<fluidThermo>(fluidThermo::dictName);

        const volVectorField& U = lookupObject<volVectorField>(fieldName_);

        return store
        (
            resultName_,
            mag(U)/sqrt(thermo.gamma()*thermo.p()/thermo.rho())
        );
    }
    else
    {
        return false;
    }
}
```

Note how different the code above which obtains a reference to the velocity
field looks compared to the bit of code from the solver that actually reads the
velocity field from disk. In post-processing, after a case has been run, the
function object could do the same: simply read the velocity for each time step.

However, without the object registry all the run-time magic of OpenFOAM would
be sort-of impossible. Simply reading the velocity wouldn't do, because the
solver continuously update the velocity, and the `functionObject` would have a
hard time to get hold of the current velocity data, if it weren't for the
object registry.
