# CILOSTAZOL

## Existing technologies

### GraalVM

> TODO: Overview

### Truffle

> TODO: Overview

### .NET

> TODO: Overview

### BACIL

> TODO: Overview

### Espresso

> TODO: Overview

## Problem analysis

### CIL vs. Bytecode

> TODO: diferencies in opcodes etc.

### BACIL 

> TODO: Unsufficient type system, parser, bugs etc...

### Espresso

> TODO: Diferencies between Java and C#

## Solution

The solution consists of many parts responsible for distinct purposes. 
We provide brief descriptions of them to make the navigation between them easier.  
The project solution contains four modules:

- **cil-parser** - It contains a low-level parser of cil metadata which is not dependent on the rest of the interpreter. Basically, It provides API for navigating through CIL meta tables.
- **language** - It is the core of the interpreter. It contains a definition of cil symbols like *class* or *method*, an object model holding user data, nodes representing cil code, factories using the mentioned parser yielding the symbols, static analysis of types, a context holding several caches, and tests testing metadata representation.
- **launcher** - It is a launcher of the interpreted language.
- **tests** - It contains a custom framework for testing end-to-end tests taking *.cs* sources, compiling them, executing them in the interpreter, and asserting the results. 

> Overview of CILOSTAZOL project architecture

![architecture_overview](./img/CILOSTAZOL.png)

Although the detailed description of interpreting CIL will be given later, we also provide a brief overview of the pipeline to make understanding each part of the process easier.
We compute everything lazily in CILOSTAZOL, however, we use the arrows in the picture in the opposite direction to indicate data flow.
So, when the request for execute cil code arrives, we start to locate the required *.dll* files. Since we have them, we use **symbol factories** to transform the files into application data. The factories use **low-level parser** to obtain meta tables and streams. 
Then, it starts to assemble them into symbols that are used during the interpretation.
Because this process can take a long time, we extensively use caches located in the context.
Since we have the necessary symbols, we find an assembly entry point and ask the execution node to execute it.
The execution node is created by a custom method, which collects necessary info about the method.
The execution node cares about many things. It uses our static analysis which is made on the first execution of the method to determine correct versions of CIL opcodes, prepares the frame, nodeized heavily used instructions, and handles exceptions.
During the evaluation of the code, there is a need to resolve symbols referred in metadata. 
**SymbolResolver** was made to provide a unified API for it.
**GuestAllocatior** is used to create objects based on symbols.
In the end, because some methods from the standard library use unsafe code or other constructs which are not supported by the CILOSTAZOL, we provide a custom implementation of commonly used methods used in our benchmarks to be able to use them.

> Overview of .dll pipeline

![pipeline](./img/Pipeline.png)

### Parser

The parser can be divided into two parts. 
We call the first part low-level parser which handles navigation between CIL meta tables and streams.
The second part is contained in dedicated symbol factories focusing on a small part of the metadata.

#### Low-level parser

We took the low-level parser from the BACIL project and transform it into a separated module since it is independent on the remaining parts of the interpreter.
Because metadata contains lots of tables that would behave in a similar way, the code generator was used to generate Java classes according to simpler tables description given in simple format.
When the generator is run, a dedicated class for each table is created.
The table consists of rows describing a part of the metadata.
The rows are implemented as smart pointers using the iterator pattern for better usage.
The columns can be constants or other pointers to different tables or streams.
Streams contain different kinds of signatures describing other metadata or string constants. 
These signatures have to be implemented manually because of harder parsing.
The signatures are part of the low-level parser as well.

We noticed some bugs in the table descriptions, which we fixed according to the ECMA specification.
There was also an issue regarding the low-level interpretation of indices, which was fixed as well. 
In the end, we reimplemented the signatures since the former implementation just parses a necessary part of the info required in BACIL.

#### Symbol factories

The symbol factories are responsible for interpreting data obtained from the low-level parser and transforming them into symbols described later.
We don't see a parallel part in the BACIL, because it was strongly connected with BACIL's type system.
We think that this architecture was wrong because of future maintainability and code extensibility.
So we separated symbol representation and creation by providing factories for each type of symbol.

Because we don't need to parse every method and class in the assembly to evaluate simple code, we use lazy evaluation of metadata, which would take a long time.
For example, we parse only referenced methods.

For performance reasons, we cache already created symbols in the context and reuse them when it is referenced again in the CIL.
We have several types of caches for different types of symbols.
The context contains separated caches for generic types, instantiated generic types, arrays, and generic method instantiations.
`NamedTypeSymbol`s have caches for defined methods and fields.
Because of the compressed design of cil metadata, we also use several indices in `ModuleSymbol` to help resolve symbols from caches.
To be sure that we always use already cached symbols, we use `SymbolResolver` which is responsible for handling all types of metadata references and returning appropriate symbols.
Since we use the `SymbolResolver` only, there is just one option, how the `NamedTypeSymbol`, `AssemblySymbol`, or `MethodSymbol` is created. The context is the only one, which calls further methods for creating these symbols when it is not found in the caches. Except for non-instantiated methods, which are created lazily and cached in the `NamedTypeSymbol`.  
More info about `SymbolResolver` can be found in the type system section.

### Type system

In this section, we describe the system of symbols heavily used in CILOSTAZOL to represent CIL components. We also talk about how cil application data are represented during the runtime. 

#### Symbols

Symbols can be divided into three categories:

- **Method-related symbols** - Red-colored symbols in the picture
- **Type-related symbols** - Yellow-colored symbols in the picture
- **.dll related symbols** - Blue-colored symbols in the picture

> Overview of symbols

![symbols](./img/TypeSystem.png)

All symbols have a common predecessor, `Symbol`. 
For testing purposes, the symbol currently contains only one method used to get the `CILOSTAZOLContext`. The reason is given in the tests section.

`TypeSymbol` contains several additional data related to types. 
It contains the info on how it is represented on the stack and inside the constructed type (meaning `class` or `struct`).
It is also required a predecessor of type, which can be used as an input into `TypeMap` mentioned later.
It also provides API for determining the assignability of CIL types.

`ReferenceTypeSymbol` represents managed pointers in CIL which consist of information about the actual location of the pointed entity (local variable, argument, object field, or array element), and the actual type of pointed object.

`ArrayTypeSymbol` describes CIL arrays.

`NamedTypeSymbol` describes named types in CIL including generic ones. It consists of other symbols for fields or methods.

On the other side, we have `MethodSymbol` which can be executed. 
The ancestors of that symbol are described in the following section.
It consists of other symbols for parameters or exception handling, which is basically a table of exception handlers containing info about the protected sections and etc...

The last group of symbols represents high-level cil containers.
`ModuleSymbol` is responsible for creating `TypeSymbols` defined locally.
`AssemblySymbol` is responsible for creating `TypeSymbols` defined in their modules.


#### Generics

The main challenge was dealing with generics.
We had to think of a mechanism for instantiating generic types.
We got inspiration from Roslyn and use the following observation.
Generic entities can be found in three states: opened, substituted, and constructed.
An entity is opened when it is generic and no instantiation has been done yet.
An entity can become substituted when it contains another entity (excluding type arguments) which is a type parameter not belonging to the containing entity.
An entity becomes constructed when it is instantiated by types.

> You can see the examples below.
>
> ```csharp
> class A<Ta> 
> {
>   void Foo(Ta p1) {}   
> }
> class B<Tb> : A<Tb> {} // A<Tb>.Foo(Tb p1) is a substituted method.
> ```

This observation led to the creation of three types of method symbols. MethodSymbol represents an opened generic entity. The `SubstitutedMethodSymbol` represents a substituted entity and the `ConstructedMethod` symbol represents the last option.

We didn't make a `SubstituteNamedTypeSymbol` because we don't support nested classes.

Instantiation of generic entities is done by `TypeMap`.
When we instantiate a type or method, we create a type map that maps type parameters to provided type arguments.
In the case of the substituted method, we provide this type map of constructed defining type to the `SubstitutedMethodSymbol`.
When we want to find out entities of constructed types of methods, we use this map to map the entities contained in the map.

#### Static object model

> TODO

#### Strings

> TODO

#### Arrays

> TODO


### Interpreter

> TODO:
> > - Context
> - SymbolResolver
> - Execution
> - Nodeization
> - Exceptions
> - OSR
> - Loading strings
> - References
> - Static analysis
> - Extern umnanaged code
> - STDLIB

### Launcher

> TODO:
> description
> cmd line args

## Tests

### Own tests

> TODO: our framework, table with tested features ?

### Benchmark game

> TODO: statistics

## Apendix

> TODO: How to run it
