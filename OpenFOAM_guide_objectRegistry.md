---
author: OpenFOAMWiki
title: OpenFOAM guide/`objectRegistry`
---

# OpenFOAM guide/`objectRegistry`


The `objectRegistry` is a hierarchical database that OpenFOAM uses to organize
its model-related data. It is complemented by `IOobject`, and `regIOobject`.
`IOobject` is a class that provides standardized input / output support, as
well as giving access to `runTime`, the root of the `objectRegistry`.
`regIOobject` automatically manages the registration and deregistration of
objects to the `objectRegistry`.


## Contents
1. [Why is it needed?](#why-is-it-needed)
2. [When do you need it?](#when-do-you-need-it)
3. [How do you use it?](#how-do-you-use-it)
4. [How does it work?](#how-does-it-work)
    1. [Overview](#overview)
    2. [Hierarchy](#hierarchy)
        1. [Hierarchy in-memory](#hierarchy-in-memory)
        2. [Hierarchy on-disk](#hierarchy-on-disk)
    3. [Read / write functions](#read-write-functions)
    4. [IOobject versus regIOobject](#ioobject-versus-regioobject)


## Why is it needed?

It is needed to simplify the communication between the _solvers_ and their
data.  OpenFOAM implements many modelling solution methods such as:

- finite volume method;
- finite differencing method;
- finite area method; and
- Ordinary Differential Equation solvers.

It achieves this through _abstraction_ and _templating_ - or in other words -
by chopping up its code into many small pieces that can be used by all these
solution methods. To make this possible, all common elements of the modelling
data are wrapped up into a single universal framework. This is the
`objectRegistry`.


## When do you need it?

Anytime you are working with modelling data. For example, this
includes:

- physical models;
- field data;
- dictionaries; and
- the mesh itself.

It is hard to imagine a situation where you will not need it.


## How do you use it?


See the "_Input/Output operations using dictionaries and the IOobject class_" article.


## How does it work?

### Overview

The main components of the `objectRegistry` are:

- the `IOobject`;
- the `regIOobject`; and
- the `objectRegistry`.

The `objectRegistry` is a _hash table_ with a few additional member variables
and methods. It catalogues `regIOobject` pointers by `IOobject::name_`, and
helps manage their _read_ and _write_ operations. Every object it catalogues
exists _in-memory_ and usually _on-disk_ in some form, the nature of which is
governed by _read_ and _write_ options carried out by the individual
`regIOobjects`.


### Hierarchy

There are actually two hierarchies in effect with the `objectRegistry:`

- the hierarchy of objects held `in-memory;` and
- the hierarchy of objects `on-disk.`


#### Hierarchy in-memory


An `objectRegistry` is itself a `regIOobject`, which means it can be a
registered member of another `objectRegistry`. This leads to hierarchies. For
example, `scalarTransportFoam` creates this `objectRegistry` hierarchy when it
initializes (in order of appearance):

```
runTime                 //Time              (objectRegistry)
|-controlDict           //IOdictionary      (regIOobject)
`-mesh                  //fvMesh            (objectRegistry)
 |-fvSchemes            //IOdictionary      (regIOobject)
 |-fvSolution           //IOdictionary      (regIOobject)
 |-points               //pointIOField      (regIOobject)
 |-faces                //faceIOlist        (regIOobject)
 |-owner                //labelIOlist       (regIOobject)
 |-neighbour            //labelIOlist       (regIOobject)
 |-boundary             //polyBoundaryMesh  (regIOobject)
 |-pointZones           //pointZoneMesh     (regIOobject)
 |-faceZones            //faceZoneMesh      (regIOobject)
 |-cellZones            //cellZoneMesh      (regIOobject)
 |-T                    //volScalarField    (regIOobject)
 |-U                    //volVectorField    (regIOobject)
 `-transportProperties  //IOdictionary      (regIOobject)
```

- In-memory hierarchy of `scalarTransportFoam` `objectRegistry`, in order of
  appearance.

Every `objectRegistry` has:

- pointers to all its child `regIOobjects`;
- a reference to its parent `objectRegistry`; and
- a reference to the master `objectRegistry`;
  - the master is always `runTime`.


#### Hierarchy on-disk

The _on_disk- hierarchy depends on the _read_/_write_ behaviour of the objects.
`objectRegistry` objects have different _read_ and _write_ behaviour than
"regular" `regIOobjects`. During a typical _write_ operation, a `regIOobject`
only has to _write_ its data to a file, whereas an `objectRegistry` cycles
through all its child objects, and tells them each ot _write_ themselves.
_Read_ operations are performed on an _as-needed_ basis.

A `regIOobject` writes its file to

```
instance_/local_/name_
```

where:

- `instance_` can be:

  ```
  * timeName()        normally [caseDirectory]/[timeName]     e.g. reactor/0.045
  * system()          normally [caseDirectory]/system         e.g. reactor/system
  * constant()        normally [caseDirectory]/constant       e.g. reactor/constant
  * caseSystem()      normally [caseDirectory]/system, but becomes ../system in parallel runs
  * caseConstant()    normally [caseDirectory]/constant, but becomes ../constant in parallel runs
  ```

  - Functions above are given local to `runTime`

- `local_` can be anything, and is _optional_; and
- `name_` can be anything.

Similarly, _read-only_ files placed in any of these paths can be read by the
solver, provided it knows to look for it. The _on-disk_ hierarchy written by
`scalarTransportFoam` is:


```
caseDirectory           // directory
|-system                // directory
| |-controlDict         // read-only
| |-fvSchemes           // read-only
| `-fvSolution          // read-only
|-constant              // directory
| |-transportProperties // read-only
| `-polyMesh            // directory
|   |-boundary          // read-only
|   |-faces             // read-only
|   |-neighbour         // read-only
|   |-owner             // read-only
|   `-points            // read-only
`-[timeName, eg 1.5]    // directory
  |-uniform             // directory
  | `-time              // read / write
  |-phi                 // read / write
  |-T                   // read / write
  `-U                   // read / write
```

- _On-disk_ hierarchy of `scalarTransportFoam` `objectRegistry`.



### Read / write functions


The `objectRegistry` employs `virtual` functions to achieve a universal framework.
When `runTime.writeNow()` is called, this is approximately what happens:

1. `runTime` calls `regIOobject::write()`
2. `regIOobject` calls the `virtual` function `writeObject.` Normally this would call
   `objectRegistry::writeObject`, but Time has overriden this:
3. `runTime::writeObject` first writes an `IOdictionary` called time to the
   `[caseName]/[timeName]/constant` directory before calling the regular
   `objectRegistry::writeObject`
4. `objectRegistry::writeObject` cycles through all its child `regIOobject`s and
   calls `writeObject` on each of them.
5. If the child object is a "regular" `regIOobject`, `regIOobject::writeObject`
   will be called.
6. `regIOobject::writeObject` will open the `OFstream` and call `writeData`, a
   function that must be defined in all `regIOobject`s.


Important write functions are:

1. `writeObject`:
  - `objectRegistry` uses this to call `writeObject` on all its child
    `regIOobject`s
  - `regIOobject` uses this to call `writeData` on itself
  - Some derived `objectRegistry` classes override this one, such as `Time`.

2. `writeData`:
  - `objectRegistry` implements this as an error-throw. Should not be called.
  - `regIOobject` purely `virtual` function (i.e. `= 0`). All derived classes
    _must_ override this.


Important read functions are:

1. `objectRegistry` can scan the case directory to find any modified files that
need to be reread. It accomplishes this with:
  - `readIfModified`; and
  - `readModifiedObjects`.

2. `regIOobject`
  - `read` - opens file stream with `readStream`, reads data with `readData`,
    closes the file stream;
  - `readData` - must be overridden by the derived class, otherwise returns a
    _fail_;
  - `readStream` - opens file stream and checks its type.

`IOobject` handles other _read_ and _write_ operations such as reading and
writing the _header_.


### IOobject versus `regIOobject`

An `IOobject` can be thought of as a dormant `regIOobject`. It doesn't have
_data_ input / output, and it doesn't automatically register / deregister
itself from its `objectRegistry`. In fact, IOobjects do not really exist -
there isn't a single `IOobject` class that isn't also a `regIOobject` (apart
from `IOobject` itself). The separation between these two objects only comes
into play when passing them as parameters:

- objects are passed as `IOobject` almost exclusively; whereas
- `regIOobject`s are passed only to register themselves in the `objectRegistry`
  (and in the constructor parameters for themselves).


An `IOobject` is a class that gives an object all the information that
`objectRegistry` needs to catalogue it. This includes:

- `name_` - this serves as both:
  - its _filename_ on the hard drive; and
  - its _key_ in the `objectRegistry` hash table;
- `instance_` - the path to its _file_, not including the final backslash;
- `local_` - an optional local path;
- `db_` - the `objectRegistry` with which it will register or has registered;
- `rOpt` - its _read_ option; and
- `wOpt` - its _write_ option.


On the other hand, a `regIOobject` is an `IOobject` that also automatically
registers and deregisters itself from its parent `objectRegistry`. It achieves
this with its constructors and its destructor.
