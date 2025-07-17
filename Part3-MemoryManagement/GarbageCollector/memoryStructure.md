## The Spur Memory Manager Overview



The Pharo virtual machine implements an object memory manager named Spur.
An object memory manager is a memory manager whose allocation units are objects.
In contrast to the operating system memory manager that manipulates raw memory, the Spur memory manager manipulates only objects.
For example, the lowest-level allocation operation is to allocate an object specifying the desired number of slots, format and class index.
In the previous chapters, we saw the object format and what these three arguments mean.

```smalltalk=true
memoryManager
  allocateSlots: numberOfSlots
  format: instanceSpecification
  classIndex: classIndex
```


The virtual machine tracks the life-cycle of all objects it allocates.
The memory manager implements an automatic garbage collection mechanism.
It detects when an object has no more incoming references, and deallocates it.
The garbage collector of Spur is precise and generational.
It is precise because it distinguishes non-ambiguously object pointers from random memory addresses.
It is generational because it categorizes objects _by age_, treating them differently depending on their age.

In this chapter we give an overview of the Spur memory manager, and the concepts behind it.
We will study how the memory is organized, how object generations impact this organization, and how objects grow old in this generational setup.

### Spur features


The Spur memory model supports the following features:

- Support both 32 and 64 bits.
- Performance improvement. Several decisions led to a much faster system \(new GC, large hash, immediate characters\).
- Variable sized and segmented memory. The memory allocated in the operating system by the virtual machine can grow and shrink according to the image size. Pharo images as large as several Gb are possible.
- Incremental and efficient garbage collector. As we describe in the following chapters, the GC is now.
- Fast become: the model introduces forwarders as special objects that avoid to walk the complete heap to swap references.
- Ephemerons: the model introduces an advanced weak structure called _Ephemeron_.  An Ephemeron is an object which refers strongly to its contents as long as the Ephemeron’s key is not garbage collected, and weakly from then on.
- Pinned objects. Pinned objects will not be moved by the garbage collector. This is an important point for the Foreign Function Interface - as you can read in the corresponding book.

### Memory Structure Overview


The Spur memory manager layouts its memory in two main sections: the new space and the old space.
The new space contains objects considered young i.e., objects that have been recently created.
The old space contains objets that did survive in the new space for some time, and were promoted as adults in the old space.

At startup, the memory manager requests the operating system a chunk of raw memory to store the new space and the old space.
The memory manager uses the first part of this memory as the new space, and the rest as old space.
Figure *@memoryMap@* depicts how the two spaces are laid out in memory, considering that lower addresses are at the left, and higher addresses are at the right.

![Memory Map: a new space followed by an old space.](figures/memoryMap.pdf width=100&label=memoryMap)

Addresses in the new space are lower than those in the old space.
This way, the VM easily determines if an object is old or young by comparing its address against the limit of the new space.
The memory manager stores the limits of the new space as `newSpaceStart` and `newSpaceLimit`. It defines that an object is young if its address is less than the new space limit. See Listing *@youngobject@*.

```smalltalk=true&caption=A young object is an object located below the newSpaceLimit.&anchor=youngobject
memoryManager newSpaceStart.
memoryManager newSpaceLimit.

SpurMemoryManager >> isYoung: oop
	<api>
	"Answer if oop is young."
	^(self isNonImmediate: oop)
	  and: [self oop: oop isLessThan: newSpaceLimit]
```


### Memory Growing and Segments


The new space remains fixed once initialized i.e., it does not grow after its allocation.
On the contrary, the old space is organized in one or more memory segments, and it can grow dynamically by adding new segments to it.
As we have seen above, the new space and the first segment of the old space are allocated in a single contiguous chunk of memory.
Newly added segments do not require to be contiguous, but they need to be at higher addresses than the first segment.

When a new segment is added, a bridge is added to the end of its previous segment.
A bridge is a fake object that fills the gap between the two segments.
Bridge objects have the format of a byte array simulating a size equals to the gap between the two segments.
They give the Spur memory manager and its garbage collector the illusion of an old space made of a single contiguous chunk of memory.
Bridge objects are not visible from the program and do not move during garbage collection.


### Memory Initialization


When the virtual machine starts, it requires memory from the operating system to store both the new space and the first segment of the old space.
The size of the new space is computed from a parameter stored in the image file header.
The image file, storing all objects in previous sessions, is loaded into the first segment of the old space.
The size of this first segment is computed as the addition of the image size and a free space headroom to fit objects coming from the new space.

### Spur Generational Garbage Collection


Spur implements a generational automatic garbage collector based on an heuristic named the _generational hypothesis_.
The generational hypothesis states that most objects die young, especially true in highly-interactive applications, so young objects are stored separately from old objects.
This is why the Pharo VM uses two different garbage collector algorithms: one for new objects implementing a generation scavenger, and one for old objects implementing a mark and compact.

By default, the memory manager allocates objects in the new space.
When the new space has little space left, it is garbage collected using a copy collection algorithm named generation scavenger, that we will explore in detail in a following chapter.
The new space is much smaller than the old space, so garbage collecting it is fast, producing unnoticeable application pauses.
If the generational hypothesis holds, unused young objects are reclaimed shortly after their instantiation and never moved to the old space.

Objects that are not reclaimed during a garbage collection are called _survivors_.
As objects survive several new space garbage collections, they grow old.
Eventually, objects old enough are _tenured_: they are moved to the old space.
The old space is several times bigger than the new space, thus garbage collecting it is expensive and creates long application pauses.
Most objects are collected during new space collections, so collecting the old space is not often required.

When the old space has little space left, a mark and compact collection algorithm reclaims unused objects. This algorithm first marks all used objects, and then scans the entire old space freeing unmarked objects and compacting the memory.

### The Stack

The stack \(both Pharo and C\) resides in the lower address of the memory.
This is the stack used by the C code and also the stack fames are allocated in this stack.
All the execution of a process stores the information in the stack.
The stack is the real representation of the contexts in the image.
The frames are in a sequence in the stack.
Each frame knows the calling frame with a pointer.
Objects referenced into stack frames are retained i.e., never garbage collected.

### Conclusion


We sketched a first overview of the memory architecture of Pharo.