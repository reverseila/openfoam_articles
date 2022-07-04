---
title: The objectRegistry in OpenFOAM
author: Mustafa Bhotvawala
---

# The `objectRegistry` in OpenFOAM

You often see `IOobject` in OpenFOAM code. As the name suggests, this can read in
and write out data – read and write fields, read from dictionaries, the whole
thing. These `IOobject`s are registered into an `objectRegistry`. What we usually
see is this:


```c++
IOobject
(
    const word & name,                  // Name of what to read
    const word & instance,              // Where to read it from
    const objectRegistry & registry,    // What objectRegistry it is a part of
    readOption r = NO_READ,             // Does it read?
    writeOption w = NO_WRITE,           // Does it write?
    bool registerObject = true
)
```


As seen above, the comments cover most of what you need to know – but what is
the `objectRegistry`? The object registry is a _hash table_ which catalogues
all the `regIOobjects` and helps manage their read and write operations. The
top level `objectRegistry` associates with the `Time` class. This means that
each Time object has a separate list of entities that it controls. This means –
when your simulation starts, a time object is created and then everything that
goes into your simulation – your mesh, your solver settings and discretization
schemes – they link to that time object . To do this, Time creates an object
which follows the simulation in terms of time-steps. This includes a start
time, an end time, and a time step size.


```
runTime                 // Time              (objectRegistry)
|-controlDict           // IOdictionary      (regIOobject)
`-mesh                  // fvMesh            (objectRegistry)
 |-fvSchemes            // IOdictionary      (regIOobject)
 |-fvSolution           // IOdictionary      (regIOobject)
 |-points               // pointIOField      (regIOobject)
 |-faces                // faceIOlist        (regIOobject)
 |-owner                // labelIOlist       (regIOobject)
 |-neighbour            // labelIOlist       (regIOobject)
 |-boundary             // polyBoundaryMesh  (regIOobject)
 |-pointZones           // pointZoneMesh     (regIOobject)
 |-faceZones            // faceZoneMesh      (regIOobject)
 |-cellZones            // cellZoneMesh      (regIOobject)
 |-T                    // volScalarField    (regIOobject)
 |-U                    // volVectorField    (regIOobject)
 `-transportProperties  // IOdictionary      (regIOobject)
```


How would a dictionary like `transportProperties` be read into OpenFOAM?


```c++
IOdictionary transportProperties
(
     IOobject
     (
          "transportProperties",  // Name of what to read
          runTime.constant(),     // Where to read it from
          mesh,                   // What objectRegistry it is a part of (mesh)
          IOobject::MUST_READ,    // Does it read? (yes)
          IOobject::NO_WRITE      // Does it write? (no)
     )
);
```


`transportProperties` is present in the `constant` folder, hence the instance
field is populated by `runtime.constant()`.


In addition to constant(), instance can be:

- `timeName()` , where `runtime.timeName()` would read/write from each
  individual time-step, for example, `pitzDaily/0.01`.

- `system()`, where `runtime.system()` would read/write from the `system`
  folder, for example, `pitzDaily/system`.



Reading a `volScalarField` pressure from the disk goes like this.


```c++
volScalarField p
(
    IOobject
    (
        "p",
        runTime.timeName(),
        mesh,
        IOobject::MUST_READ,
        IOobject::AUTO_WRITE
    ),
    mesh
);
```
