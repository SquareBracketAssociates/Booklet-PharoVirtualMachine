## Code Layout

The input presentation is here: 
[https://rmod-files.lille.inria.fr/Team/Presentations/VMPresentations/2020-StructureOfMachineCodeMethods.pdf](https://rmod-files.lille.inria.fr/Team/Presentations/VMPresentations/2020-StructureOfMachineCodeMethods.pdf)

### Code Layout Overview

A jitted method is structured in three parts as shown in Figure *@methodStructure1@*: 

- a meta data / header
- a method body
- a meta data about the compiled code

![ Structure of a method. %width=20&anchor=methodStructure1](methodStructure1.png)

Let us go over each of such parts.

### Meta-Data Header
This header is common to Cog methods, Polymorphic inline Caches, and megamorphic call sites.

An object header at the address of a jitted method describes information about the method. 
The method header consists of (see Figure *@methodStructureHeaderWithZoom@*):

![ Structure of a method header. %width=50&anchor=methodStructureHeaderWithZoom](methodStructureHeaderWithZoom.png)

- _A header_. It makes jitted method objects similar to normal objects, marked as reachable so as to survive garbage collection until it is not needed to free up space. This is needed since jitted methods are stored in a 1.4 Mb memory region.
- _Some flags_. For example to describe if we should create a frame or not (to be investigated)
- A _block size_. It expresses the size in bytes of the entire method, including the header.
- A _block entry offset_. It is the offset from the header address to the start of the method.
- A _pointer to the compiled method_ it is compiled from. Note that the compiled method has also a pointer to its jitted method. 
- The _compiled method’s header_. It is stored in the jitted method with the following attributes:
  - Flags about the type of encoder used.
  - if it has a primitive number of arguments passed to the method.
  -  number of temp variables.
  -  number of literals.
  -  frame size.






### Preamble and Postamble

The _method body_ has the following separated and tagged structure:

1.  The _abort routine_ code. It is executed if sent message fails or the stack limit is reached. 
2.   _Checked entry point_ (also called type guard). It verifies the receiver is of the expected class (receiver class check),  If there’s a mismatch between the types, the method execution is aborted.
3. _Unchecked entry point_. This entry point skips the receiver class check.
4.  For primitives, their code is next.
5.  If the method is frameful, the frame-building code is inserted. Note that a _frameless_ method has no creation of a frame code because its execution isn’t interrupted by a message call. A _frameful_ method, however, has to manage the call stack with stack frames storing arguments, receiver and interruption points’ program counter to deal with eventual message sends in their execution context.
6. _Method code_.  The actual code of the method is next.

7. _Method map_. At the end of the cog method is the method map, which has meta data identifying interesting points in the machine code. The map is read backward starting from the blockSize offset of the CogMethod. It has object reference sends and pc-mapping points in the machine code, which helps the garbage-collector find and update object references, method cache flushing, conversion between byte-code and machine code PCs by looking for matching points in the map.

### Method Entry Points

The each portion of code is tagged to give direct access to it. For example:

- The abort routine can be branched-to when method code fails.
- You can skip access to the type checking by jumping to the start of the next code block. 

### CogMethod in Pharo and C 
  
Listing *@jitmethodclass@* shows the class `CogMethod` in Pharo that represents a jitted method as shown in Listing *@jitmethodstruc@*. This is this class that once generated in C as C-structure represents jitted methods. 

```caption=Pharo class representing jitted method&anchor=jitmethodclass
VMStructType << #CogMethod
	slots: {
			 #objectHeader .
			 #homeOffset .
			 #startpc .
			 #padToWord .
			 #cmNumArgs .
			 #cmType .
			 #cmRefersToYoung .
			 #cpicHasMNUCaseOrCMIsFullBlock .
			 #cmUsageCount .
			 #cmUsesPenultimateLit .
			 #cbUsesInstVars .
			 #cmUnusedFlags .
			 #stackCheckOffset .
			 #blockSize .
			 #picUsage .
			 #methodObject .
			 #methodHeader .
			 #selector };
	sharedPools: { CogMethodConstants . VMBasicConstants . VMBytecodeConstants };
	tag: 'JIT';
	package: 'VMMaker'
```

Once the code is generated we obtain the following C structure in the file CogMethod.h (See Listing *@jitmethodstruc@*).


```caption=Jitted method C structure&language=C&anchor=jitmethodstruc
typedef struct {
	sqLong	objectHeader;
	unsigned		cmNumArgs : 8;
	unsigned		cmType : 3;
	unsigned		cmRefersToYoung : 1;
	unsigned		cpicHasMNUCaseOrCMIsFullBlock : 1;
	unsigned		cmUsageCount : 3;
	unsigned		cmUsesPenultimateLit : 1;
	unsigned		cbUsesInstVars : 1;
	unsigned		cmUnusedFlags : 2;
	unsigned		stackCheckOffset : 12;
	unsigned short	blockSize;
	unsigned short	picUsage;
	sqInt	methodObject;
	sqInt	methodHeader;
	sqInt	selector;
 } CogMethod;
```

### A word about `cmUsageCount`

Since the code zone is not infinite, it needs to be compacted from time to time. 
The field `cmUsageCount` uses 3 bits to represent the number of uses of the jitted method that is used>
It does so by walking the stack. This field is used to decide which jitted methods can be removed from the code zone.  




### Primitive Method Layout

Primitives have few variations from the normal methods, they have a C implementation or native machine code, based on these implementations, we have three types of primitives: (which ones)

- A function written in C, also called _plugin_.
- A function written completely in machine code, called _Complete primitive_.
- (is this the third one?)First part of the method body has machine code for fast paths, if the primitive fails execution, in other words the case calling the primitive doesn't correspond to a fast path, it continues through the method to delegate to a C function.
    
When The primitive code succeeds, it returns a result before having to create a new frame for the fallback code.

Complete, Unfailing and Unimplemented primitives




