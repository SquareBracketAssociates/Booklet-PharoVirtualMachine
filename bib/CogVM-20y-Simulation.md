pmissech <pierre.misse-chanabier@inria.fr>
	
mar. 28 janv. 10:40 (il y a 3 jours)
	
À lse-vmvm

Hi everyone,

There's an attempt at a summary for the article Two decades of Smalltalk VM Development written by Eliot Miranda/Clement Bera/ Elisa Gonzales Boix/Dan Ingalls, published at VMIL 2018.

This article is mostly difficult for me to understand and explain well, although I think I understand from very far away what is explained. Hope this summary will be understandable :)


The aim of this article is to say:
-Simulation rocks, come try it!
-Snapshot rocks, come try it!
-Simulation + Snapshot is incredible, come try it !

This article explains how the VM is written compiled, and the concept of snapshot which is crucial to have an interesting simulation.
It then shows how this enable simulation of the virtual machine through examples, and why this is a valuable property.


They highlight the differences between simulation and production mode:
1 - Memory architecture
2 - Runtime simulation specificity
3 - Machine code debugging

1 - Memory architecture

They explain how memory architecture is different between simulation and production mode.
I don't remember it having an impact anywhere else in the article, and the sketch provided in the article sum it up nicely.

it is mostly structural differences.


2 - Runtime simulation

First, they explain the polymorphism system that exists only during simulation.
When translated to C, polymorphism is removed, and the VM is configured to use a version of a feature (for example, which garbage collector/interpreter you want).
A translated VM is therefore configured for a specific case as a whole.
Having polymorphism helps with modularity, as well as debugging during simulation.

The simulation is written to be deterministic which means that:
- "simulated memory is not subject to address space layout randomization" which is (afaik) an OS feature to limit possibility of attacks on softwares.
- "a Synthetic clock is used so that time advances in lock  step with code execution" which I suppose is to have a synchronization between the threads used by the VM.

Jit simulation is done by FFI calls to bindings with processors simulators (Bochs/Skyeye...).
The simulators receive a smalltalk ByteArray, and the smalltalk register state which is accessed through external memory access (Alien).
I didn't understand the part of Smalltallk and machine code interactions well enough to explain it.

Next part explains how to debug machine code, but I didn't understand it well enough to explain it either/


The experience report section show it by explaining how simulation + snapshot have been used to develop and debug a new Garbage collection, the JIT.

The first example describes that codding in the debugger was very practical to implement new bytecodes template for the JIT.

Second one explains how lemming debugging was used with the simulation to create snapshots of buggy cases for the garbage collector.

Process is as follow:
*GC about to start*
Copy entire memory
*GC On copy of memory*
memory isCorrupted
ifTrue: * snapshot *
ifFalse: * resume execution *

I assume some stuff in this process, it's not described as clearly (to me) in the paper.
Point of this debugging technique is to create reproducible cases more easily, and automatically.


Next they explain how a bug in a bytecode JIT compilation was dealt with.
They were creating snapshots of points in the simulation where they knew the problem happened, to find the previous point in the execution where the problem may come from.
By creating new snapshots at different points in the simulation, they were able to avoid simulation start up, and to use smalltalk tools to figure it out.
Finally they configured the processor simulator to step through machine code, and used conditional breakpoints to detect when the field was written.

The next case shows quickly that since the code is written in smalltalk, they were able to run the jit in smalltalk on small cases to develop it incrementally without compilation pauses.

The last two experiment they report show that simulation provides a nice way to create statistics, and how statistics can direct implementation decisions.
They launch simulation up to one arbitrary point (in that case when machine code used by the JIT was 1 MB) and gather statistics.
When thinking about implementing an optimization, they noticed that this optimization was done prematurely in 17% of the cases. this therefore allow them to tune the optimization.
They also mention that this was done by other peoples, and that where they did it an hour, it took several days for the others.


Finally they show limitation and related work

The first limitation is performance.
A bug happening 15 minutes after startup would take 50 hours to simulate.
This is amortized by the creation of snapshots right before the bug happens.

Call to external code can be necessary for development of specific stuff such as the file management.
and there is several bugs in FFI.

They explain quickly that the simulator is also used for the bootstrap, conversion of images from 32 to 64...

Finally in the related work

Graal -> discarded because they apparently don't have articles on VM development tools.
Maxine ->meta circular vm no production mode. All they have is therefore a simulation.
Also note that most production vm are still compiling through the C/C++ compiler.
RPython -> Restrictive pythin, which is close to python than Slang is to smalltalk.
- Slow compilation (40 minutes vs seconds for slang)
- Slow simulation. Since they don't have snapshots, they have to simulate everything till the bug occurs every time.


Conclusion: Smalltalk, simulation, snapshot, live programming [...] is cool !


