pmissech <pierre.misse-chanabier@inria.fr>
	
jeu. 23 janv. 10:32 (il y a 5 jours)
	
À lse-vmvm
Hi,

I'm reading Garbage Collection: Algorithms for automatic dynamic memory
management.
And I was asked to share my reading, so here we go :)


There is three classic algorithms

1 - Reference counting
2 - Mark sweep
3 - Copying

For each of them, I will give an overview of the algorithm, and its
simplified forces and weaknesses.


1 - Reference counting

Every object of the system, has an additional 1 header representing the
number of references to it.

Basically:

| a b |
a := Object new. "references count (object) = 1"
b := a. "references count (object) = 2"
a := 1."references count (object) = 1"
*sortie de stack frame*
"references count (object) = 0,  nothing knows this object anymore, it
is therefore given back to the available memory"

Every time a reference to this object is created, the counter is
incremented.
Every time a reference to this object is removed, the counter is decreased.
If the number of references reaches 0, the memory is given back to the
available memory.

+ No break of computation, the cost of garbage collection is spread on
the runtime execution.
+ There is no garbage in memory (except point #5).
- An additional field for each object.
- Costly write / free (has to decrement its fields pointers, recursively).
- Cannot reclaim cycles.
- heap fragmentation.

Let's see an example of that last one

| a |
a := RandomObject new. "references count (randomObject) = 1"
a fieldPut: a."references count (randomObject) = 2"
*sortie de stack frame*
"references count (randomObject) = 1"

The References a to the objects in memory is destroyed at the end of the
frame but a references itself, the reference counter does not reach 0,
memory is not reclaimed


2 - Mark sweep (current old space garbage collection, if i'm not mistaken

Every Object in the system has one bit for garbage collection.
When the memory is exhausted, it performs a garbage collection, then
resume the execution of the program.
The GC operation is separated in two phases: Mark, and Sweep (Crazy, I
know :p)
The mark operation will take every reference to objects available from
variable, known as roots.
It will walk through the object graph they refer to, and mark them with
the GC bit.
Then, the sweep operation will walk through the complete heap, and
reclaim what is not marked.

Basically:

| a |
a := RandomObject new.
a field: Something new.
a field: Anything new.

Before garbage collection we have 3 objects in memory: aRandomObject,
aSomething, anAnything.

*Mark starting*
a is a variable that knows an Object, so we walk through its object graph.
*mark aRandomObject*
*walk aRandomObject*
*mark anAnything*
*walk anAnything*
*Mark done*

2 objects have been tag as in use during this phase.

*Sweep starting*
*is first memory chunk marked? => yes*
*is second memory chunk marked? => no => add to available memory*
*is third memory chunk marked? => yes*
*Sweep done*

The two marked object have survived the garbage collection, the unmarked
one has been reclaimed.

+ reclaim cycle easily.
~ one extra bit in header.
- Stops program execution
- heap fragmentation.


3 Copying (current new space garbage collection)

roots pointing to objects are kept.
Uses 2 different memory space the past, and the future.
The garbage collection operation is equivalent to stopping time.
At the start of the garbage collection, every object is in the past, and
none are in the future.
When memory is exhausted the GC goes through roots and walks their
object graphs.
Whenever the garbage collection encounters an object, it copies it from
the past to the future.
In its place, it leaves a forwarder, an address that points to the
position of this object in the future.
When copying the fields of an object, it may encounter a forwarder to
object already copied in the future can be encountered.
The reference is therefore replaced with the address of the object in
the future.
Unreachable objects are ignored.
At the end of garbage collection, the future will be the past of the
next garbage collection, and the past becomes the next future (to reuse
this memory space)

Basically (ish):


GC example

| a |
a := RandomObject new.
a field: Something new.
a field: Anything new.

*GC Start*
a is a root, aRandomObject is copied in the future.
walk through a.
anAnything is copied in the future.
*GC finished, swaping past and future addresses*


Forwarder example

| b f |
f := Something new.
a := RandomObject new.
a field: f.

*GC starts*
f is a root, aSomething is copied in the future.
walk through f (nothing to do).
b is a root, aRandomObject is copied in the future.
walk through b.
b field has a forwarder in memory, in the past.
b field reference's become the current address of f, in the future.
done walking b
*GC finished*

+ allocation cost low.
+ no fragmentation in memory, it is compacted every time the GC runs.
+ easy out of memory check
+ cycles management
- twice as much memory to work



Most of those downsides can be managed (which I'll see in the next part
of the book :p), and some downsides can be seen as very costly, but may
in fact be more manageable than one would expect.
An example of that is that the Copying GC takes twice as much memory as
the mark and sweep.
But the mark and sweep will have a fragmented memory.
The size of the memory may therefore grow faster.
So the GC choice depends on the program executed !
Have a lot of fragmentation? Copying is probably better
Have a lot of objects that don't move? Mark sweep is probably better !
You cannot accept long pauses? How about reference counting?

This last statements are Extremely simplified, but it is one way of
looking at this problem, and states that there is not THE ONE garbage
collector (yet :D).
This also shows that depending on what you're running, configuring the
Garbage collection is interesting !


I hope this overview was comprehensible, and don't hesitate to ask
questions :)


Pierre