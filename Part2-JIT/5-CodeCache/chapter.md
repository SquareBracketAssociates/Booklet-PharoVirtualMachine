## The Cogit JIT Compiler

Commonly executed methods in Pharo are compiled to machine code by the Cogit just-in-time (JIT) compiler.
The Cogit JIT compiler is bytecode-to-machine code compiler in the VM, whose main objective is to remove interpretation overhead.
It implements peephole optimizations through abstract interpretation, and uses an x86-inspired intermediate representation with fixed virtual registers.
In this chapter we give a general view on the compiler architecture, its intermediate representation, the different compilation passes.

### The Cogit Architecture

### Intermediate Representation: the Cogit Register Transfer Language

### Semantic Analysis

### Compiler Front-end

### Code Generation