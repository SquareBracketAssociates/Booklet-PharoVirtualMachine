pmissech <pierre.misse-chanabier@inria.fr>
	
jeu. 30 janv. 10:46 (il y a 23 heures)
	
À lse-vmvm
Summary for The Cog Smalltalk Virtual Machine : writing a JIT in a
high-level dynamic language.

by Eliot Miranda ,VMIL 2011
http://web.cs.iastate.edu/~design/vmil/2011/papers/p03-miranda.pdf


This was a bit simpler to understand than 2 decade of Smalltalk dev, and
actually partly clears up one of the part I didn't understand in it (the
debugging machine code part).
(2 decades summary ->
https://lists.gforge.inria.fr/mailman/private/lse-vmvm/2020-January/000150.html)


In this article, it is explained how the Cog JIT (Cogit) was developed
in Slang.
It also explains attempt to show properties of developing the low level
code in a high level language.

It first explains  a bit what is slang (which I won't explain every time :p)
Before developing the JIT, the Interpreter was monolithic.
To be able to add the JIT, it was split into the Interpreter and the
Object memory.
The JIT was also kept isolated, to improve modularity.

It then introduce Botch, which is a framework for processor simulation
for x86 code.
Memory is represented by a big ByteArray.
Glue code was necessary for several operations, such as memory access
(needed to access the ByteArray).
These overriden operation are isolated in a vm plugin.
The receiver of the plugin is an Alien, a wrapper to manipulate an
external address, to the Bochs C++ objects.
The memory is an usual argument of those messages.

The jited version of a method (machine code) is stored in the ByteArray
memory.

<Not sure what he explains>

He seems to use ByteArray and memory object to differenciate between the
smalltalk and native memory, but it's used in a way that make me doubt it.
Also, he introduce the acronyms 'ADT', without any explanation (that i
noticed)

</Not sure what he explains>

<SideNote: Illegal address>

(after talking to Vince)
When a program is executed, it is given a piece of memory.
This memory is 'virtualized', so it starts at 0.
In this virtual memory space, you have addresses reserved for the OS
(I'm not clear on why).
Theses addresses are called Illegal addresses.
When attempting to use an Illegal address, the OS will raise an exception.

</SideNote>

When the simulation needs to execute machine code (jited method), it
calls a primitive interfacing with Bochs.
When the machine code needs to execute Smalltak (Not (yet) jitted method
for exemple), it needs a way to communicate with the simulation.
Every method that may need to be called from machine code is put in a
dictionary, using for key an illegal address (see sideNote).
When encountering an illegal address, Bochs throws an error that is
handled by Smalltalk code.
The simulation search the dictionary at the illegal address key, and
resume execution with that method (or block).


He then explains several small things:
- His company needed performance -> Implementation of stack to context
mapping, without having to wait for the jit.
- Cog evolved on several points.
- Advantage of using a mixed model execution, something are not needed
to be translated (class initialization for example), reducing code size.
- Cogit compile a method on its second use.
- GC may remove some method in use, which  can be tricky, is simplified
in mixed execution, because it's done only at 'suspension points' <- not
sure i get this right.
- There was an implementation of the green thread (not explained more
than a simple definition)


"Implementation of stack to register mapping code generator  that avoids
reifying operands on the actual stack by maintaining a simulation stack
during compilation and only generating code to access an operand when
compiling a bytecode that consumes that operand such as a send"
I'm quoting this one because i don't think I can reformulate it better.

- support for adaptative/speculative optimization (through polymorphic
inline caches for example).
- The bytecode set is plugable, descriptions for byteCodes are stored in
an array -> There is no monolithic switch statement in Slang code,
although one is generated (but not really used (ish)).
- Future work -> Spur? Use a class table instead of storing pointers to
class, ease class instances / inline caching space requirement.

He then proceed to explaining "clever tricks

To avoid integer overflows, he uses c and debug specific code (he gives
the #isIntegerValue: method as example).

To avoid maintaining two different simulators, he generates simulation
methods for the subclass he wants to simulate that override a method in
the superclass.