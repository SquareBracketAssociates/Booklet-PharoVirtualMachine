## Introduction: The Virtual Machine Runtime

This book explains many of the insides and interesting design points of the Pharo Virtual Machine.
The Pharo Virtual Machine is what makes possible to execute Pharo programs: it manages the execution, the memory, the access to external resources.
To do so, the path taken is to implement a so-called runtime system.
That is, it provides a set of services and components that are available at run time to execute Pharo programs.

This chapter gives an overview of the VM components and how they are related to each other.
Of course, these _many_ components will be explained in detail in _many_ subsequent chapters.
You are free to read the book from begin to end.
For those that want a more guided lecture, the last section in this chapter proposes several reading orders.

### Runtime Environment Overview

The VM is organized in many different components.
We call _static components_ to those components that exist when the VM is not executed.
Static components are shipped in VM libraries: for example, this is the case of the interpreter, the JIT compiler and the garbage collector.
We call _runtime components_ to those components that are set up or created when the VM starts.
For example, these are the memory where objects are allocated, the execution stack or the cache where native methods are stored after JIT compilation.

Figure *@fig:runtime@* shows a very high-level view of the main VM components.

![Runtime Overview.](figures/runtime-system.pdf?label=fig:runtime)

#### The Heap and the Memory Manager

At the very core of the Pharo VM there is the memory manager, the façade to the _heap_ and all object accesses in the system.
It implements two key responsibilities: object encoding/decoding and memory (de)allocation.

The _heap_ is a space of memory where objects are stored.
The memory manager is _the component_ whose main responsibility is the organization of the heap.
It dynamically allocates the memory that is part of the heap, and organizes the heap internals.
It implements the algorithms that manipulate the heap: garbage collect unreachable objects, track reachable objects, move objects around.
The current implementation implements two different strategies:
- a bump allocator with a copy collector for recently allocated objects
- a free-list based allocator with a mark-compact collector for old objects

The second key responsibility of the memory manager is to dictate how objects represented in memory.
It implements an object format called _Spur_ that specifies how many slots an object has, how much memory a slot occupies, how objects are aligned in memory, and so on. 
The memory manager provides methods that _abstract_ the rest of the system from such internal details (yes, sometimes we don't need to know how many _shifts_ and _bitand_ are required to do something).

#### An Image-based Virtual Machine

Pharo is an image-based programming language.
This means that all objects of a program can be saved into a so-called _snapshot_ or _image_, to be later restored.

On the one hand, creating a snapshot is nothing else than making a dump of the heap into a file.
All object references are stored as absolute pointers in memory.
On the other hand, restoring a snapshot means loading such a file into memory, and potentially relocating (or _swizzling_) those pointers if necessary.

#### The Image->VM Interface: the Special Objects Array

Such information is used to perform operations such as type checks, allocations of special objects, or activate callbacks such as the famous _does not understand_.
The runtime system requires, from time to time, to have specific information from the heap:
- where is the object `nil`?
- where is the class `Array`?
- where is the selector `doesNotUnderstand:`?

Such an interface is cristalized by the _special objects array_.
The special objects array is an array that references a set of special objects at _well known indices_.
The following snippet shows a couple of lines of the method `newSpecialObjectsArray` that recreates an array for such purposes.

```caption=A excerpt of the special objects array
newSpecialObjectsArray
	| newArray |
	newArray := Array new: 60.

	newArray at: 1 put: nil.
	newArray at: 2 put: false.
	newArray at: 3 put: true.
	...
	newArray at: 6 put: SmallInteger.
	newArray at: 7 put: ByteString.
	newArray at: 8 put: Array.
	newArray at: 10 put: BoxedFloat64.
	...
	newArray at: 21 put: #doesNotUnderstand:.
	newArray at: 22 put: #cannotReturn:.
	...
	^ newArray
```

The special objects array is stored with all other objects in the heap, but unlike all other objects, the VM keeps a strong reference to it.
When the VM needs to access one of these special objects, it does it through the `splObj:` method, that returns the special object at an index.
The interface is chaperoned by a set of constants specifing clear names for each constant.
For example, the following code snippet shows how the VM accesses the special objects above.

```
objectMemory splObj: NilObject.
objectMemory splObj: FalseObject.
objectMemory splObj: TrueObject.

objectMemory splObj: ClassSmallInteger
objectMemory splObj: ClassByteString.
objectMemory splObj: ClassArray.
objectMemory splObj: ClassFloat

objectMemory splObj: SelectorDoesNotUnderstand.
objectMemory splObj: SelectorCannotInterpret.
````

#### The Interpreter and the Lookup Cache

When executing Pharo code, two options are available: it's either executed by the interpreter or by compiled native code.
The interpreter is the most basic of both, and executes code that has been executed few times or cannot be compiled.

When in charge of the execution, the interpreter executes Pharo bytecode, one by one.
It implements many different optimizations both in the interpreter and in the bytecode set.
For example, common instructions are specialized and shortened, common sequences are folded, instructions are threaded.

Moreover, the interpreter contains the implementation and optimization of the method lookup algorithm, key to support message sends.
The method lookup, a method search in the receiver's hierarchy, is one of the most used and most expensives operations in Pharo programs.
To avoid the expensive lookup, the interpreter uses a lookup cache that stores the results of previous searches.
And it works pretty well.

#### The JIT Compiler and the Native Code Cache

Although the interpreter performs pretty well in modern hardware, there is a non-negligible performance overhead posed by the interpretation process itself.
Thus, commonly executed methods are compiled to machine code using a just-in-time (JIT) compiler named _cogit_, and the resulting machine code is stored in a native code cache.


#### What's not in the picture

Image loading, profiling support

### How to Approach this Book