## Code Layout

The input is here.

[https://rmod-files.lille.inria.fr/Team/Presentations/VMPresentations/2020-StructureOfMachineCodeMethods.pdf](https://rmod-files.lille.inria.fr/Team/Presentations/VMPresentations/2020-StructureOfMachineCodeMethods.pdf)

### Code Layout Overview


![ Structure of a method. %width=60&anchor=methodStructure1](methodStructure1.png)

### Meta-Data Header

A header at the address of the cog method, that describes the meta information about the method (see Figure *@ methodStructure1@*. 
It makes jitted method objects similar to normal objects, marked as reachable so as to survive garbage collection until it is not needed to free up space, which is much needed since cog methods are stored in a 1,4 mb memory region.

It consists of:

- block entry offset. 
- block size, which is the size in bytes of the entire method, including the header.
- A pointer making a circular reference to the compiled method.
- compiled method’s header is replicated in the cogged method with the following attributes:
  - Flags about the type of encoder used.
  - if it has a primitive number of arguments passed to the method.
  -  number of temp variables.
  -  number of literals.
  -  frame size.

### Preamble and Postamble

The Method body has the following separated and tagged structure:
-  the abort routine code, executed if send messages fails or the stack limit is reached. 
-  Checked entry point that verifies the receiver is of the expected class, otherwise called type Guard. If there’s a mismatch between the types, The method is aborted .
- Unchecked entry point that skips the receiver class check.
- In presence of primitives, their code is next.
- If the method is framefull, the Frame-Building code is inserted.
    - The frameless method has no creation of a frame code because its execution isn’t interrupted by a message call.
    - The frame full however has to manage the call stack with stack frames storing arguments, receiver and interruption points’ program counter to deal with eventual message sends in their execution context.
- The method code is last.

At the end of the cog method is the method map, which has meta data 
	 identifying interesting points in the machine code. The map is read backwards starting from the blockSize offset of the CogMethod. It has object references sends and pc-mapping points in the machine code, which helps garbage collector find and update object references, method cache flushing, convert between byte-code and machine code pcs by looking for matching points in the map.

### Method Entry Points

The each portion of code is tagged to give direct access to it. For example:
- The abort routine can be branched-to when method code fails.
- You can skip access to the type checking by jumping to the start of the next code block. 

### Primitive Method Layout

Primitives have fiew variations from the normal methods, they have a C implementation or native machine code, based on these implementations  we have three types of primitives:
- A function written in C, also called plugin .
- A function written completely in machine code, called Complete primitives.
- First part of the method body has machine code for fast paths, if the primitive fails executing, in other words the case calling the primitive doesn't corrrespond to a fast path, it continue through the methed to delegate to a c function.
    
When The primitive code succeeds it returns result before having to create a new frame for the fallback code.

Complete, Unfailing and Unimplemented primitives