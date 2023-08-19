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

> TODO:
> - Seperated module
> - Bugs in BACIL
> - Lazy evaluation of symbols
> - Factories

### Type system

> TODO:
> - Symbols
> - SymbolResolver
> - Context
> - Strings
> - Arrays
> - STDLIB

### Interpreter

> TODO:
> - Execution
> - SOM
> - Nodeization
> - Exceptions
> - OSR
> - Loading strings
> - References
> - Static analysis
> - Extern umnanaged code

### Launcher

> TODO:
> description
> cmd line args

## Benchmarks

### Own tests

> TODO: our framework, table with tested features ?

### Benchmark game

> TODO: statistics

## Apendix

> TODO: How to run it