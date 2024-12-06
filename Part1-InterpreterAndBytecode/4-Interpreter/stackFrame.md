## Stack Frame
@cha:stackframe

While the representation of execution as objects, _contexts_, is conceptually relevant, it has many drawbacks as mentioned in Chapter *@cha:SemanticsByExample@*. In this chapter 
we present the stack frame used by the Pharo Virtual machine. We show how the Pharo virtual machine represents the execution stack as a contiguous memory region addressing the limits of context representation. The stack frame is under the responsibility of the interpreter. This is why 
the stack manipulation are operations defined within this class. 
Chapter *@cha:interpreter@* will present in detail the functionalities of the interpreter, such as method lookup.

We first discuss the stack value representation, then we discuss the structure of stack frame elements, and finally, we show that contrary to the context representation, with a stack frame arguments are not mixed with temporaries and do not have to be duplicated between caller and callee methods. 

In a nutshell, the temporaries are managed inside the current stack frame down at lower address from the receiver, while arguments are referred to up the memory from the current frame pointer to the previous stack frame. 


### Pushing and Popping, the Stack and the Stack Pointer

The execution of Pharo code is supported mainly by a stack supporting operations such as push, pop and indexed access from the top.

The stack is a contiguous region of memory of fixed size organized in slots of one word each.
The _base_ of the stack is where the stack begins, where its oldest elements are stored.
Conversely, the _top_ of the stack is where the most recently pushed element is stored (see Figure *@stackGrowDownAddress@*).

![The stack grows down. %width=70&anchor=stackGrowDownAddress](figures/stackGrowDownAddress.pdf)


The stack is manipulated through a variable called `stackPointer`, which is a pointer to the top of the stack.


>[! important ] 
> The stack grows down: in Pharo, as in many other language implementations, the stack base is in a higher address than the stack top.
Pushing a value to the stack moves the stack pointer towards lower addresses. Popping moves it towards higher addresses.


Using the `stackPointer` variable the interpreter implements the following methods shown in Listing *@stackOps@* and *@stackOps2@*.

- `stackValue: offset` accesses values down the stack at a given offset. We see here that the values down the stack are at addresses higher than the `stackPointer` because we are adding the offset to the address of the `stackPointer`. 

- `stackTop` just returns the value at the `stackPointer`.

- `push: object` pushes a new object on the stack. We see that the stack is growing in the direction of lower addresses since the `stackPointer` address gets substracted one offset. 
Note that `sp` is just a temporary variable. 

```caption=Low-level stack operations in the interpreter&anchor=stackOps
Interpreter >> stackValue: offset
	^memory readWordAt: stackPointer + (offset * objectMemory wordSize)

Interpreter >> stackTop
	^ self stackValue: 0

Interpreter >> push: object
	memory readWordAt: (sp := stackPointer - objectMemory wordSize) put: object.
	stackPointer := sp
```

The two following operations `pop: nitems` and `popStack` follow the same logic (see Listing *@stackOps2@*). 
It should be noted that `pop:` just moves the `stackPointer` without returning the values on the stack. 

```caption=Low-level stack operations in the interpreter&anchor=stackOps2
Interpreter >> pop: nItems
	stackPointer := stackPointer + (nItems*objectMemory wordSize).
	^nil

Interpreter >> popStack
	| top |
	top := self stackTop.
	self pop: 1.
	^top
```




### First Simple Bytecode

If we assume that `fetchNextBytecode` gets the next bytecode and increases the instruction pointer, we can already understand several simple instructions such pushing well-known constants, popping from the stack, or duplicate top as shown in Listing *@firstByteCode@*

```caption=Bytecode implementing simple push and pop instructions&anchor=firstByteCode
Interpreter >> pushConstantFalseBytecode
	self fetchNextBytecode.
	self push: objectMemory falseObject.

Interpreter >> popStackBytecode
	self fetchNextBytecode.
	self pop: 1.

Interpreter >> duplicateTopBytecode
	self fetchNextBytecode.
	self push: self stackTop
```

>[! Important ] 
> Note that `popStackBytecode` just removes the top value from the stack without returning it.
> Indeed bytecode do not return values. They store them either in the stack, in temporaries or instance variables. 


### The Call Stack

Many bytecode instructions require accessing data that belongs to a particular method execution: the receiver, arguments, temporaries, or the current method to access its literals. Such an information is stored in the stack.

The stack is split in regions called _stack frames_, or frames for short.
The frames in the stack form a chain of frames, usually referred to as the _call stack_ as shown in Figure *@interpreterstack@*.
In the top of the stack there is the top stack frame representing the current method execution.

![The call stack as a linked lists of frames in the stack. %width=70&anchor=interpreterstack](figures/interpreter_call_stack.pdf)


### Stack Frame Structure

Each stack frame represents the execution of a method and has three main parts.
First, a fixed set of fields containing execution meta-data.
Second, one slot for each temporary variable in the method, initialized to `nil`.
Finally, a variable part containing the value stack where Pharo objects get pushed and popped when executing bytecode instructions.

The fixed fields in a frame are the following:
- **The caller's frame start:** a pointer used to indicate where does the caller's frame start in the stack. Used to reconstruct the call stack.
- **The executing method:** used to extract literals and other method meta-data.
- **The reified context object:** this field references the `Context` object associated with the frame, or `nil` if absent. This field will be explained in detail in the chapter about debugging support.
- **Flags:** Only available in interpreter frames (as opposed to compiled code frames). It contains a bit mask with the number of arguments, a flag indicating if the context object is set, and a flag indicating if the frame is a method or a closure execution.
- **Receiver:** The message receiver _i.e.,_ the object referenced by `self` in the current method execution.

![The structure of a stack frame and the interpreter pointers. %width=70&anchor=interpreterVariables](figures/interpreter_variables.pdf)

The frame on the top of the stack is said to be _active_.
The active frame is delimited by the variables `stackPointer` at its top and `framePointer` at its base.
While the `stackPointer` will move with each push/pop instruction, the `framePointer` points to the base of the current frame in the stack and does not change during a method execution.
Moreover, the `instructionPointer` is a pointer within the bounds of the method of the active frame.

#### Key Interpreter Pointers.

Figure *@interpreterVariables@* illustrates a key part of the interpreter infrastructure. 
The state of the interpreter is controlled by three important pointers:
- The `framePointer` which refers to the currently active frame.
- The `stackPointer` which refers to the top of the current frame stack value.
- The `instructionPointer` which refers to the next bytecode to be executed. 


### About Frame Suspension

All frames other than the active one are said to be _suspended_.
Suspended frames need the values of _their_ `framePointer`, `stackPointer` and `instructionPointer` to be stored so _e.g.,_ the stack can be traversed by the garbage collector and control can return to those frames after a return instruction.

A frame is suspended by pushing its instruction pointer to the stack before creating a new frame, and pushing its frame pointer as the first element of the next frame. Storing on each frame the frame pointer of the preceding frame transforms the stack in a linked lists of frames (See Figure *@interpreterstack@*).
Thus, the stack can be reconstructed by iterating from the top frame up to its caller's frame start until the end of the stack.

Notice that the stack pointer does not need to be stored: a suspended frame's stack pointer is the slot that precedes its suspended instruction pointer, which is found relative to its following frame.

![Creating a new stack frame main steps. %width=70&anchor=stackFrameCreation](figures/interpreter_variables_creation.pdf)


### Setting up a Stack Frame

Listing *@settingStackFrames@* presents an extract of the method `setUpFrameForMethod: aMethod receiver: rcvr`. It illustrates how a new frame is created.

- First the current frame is suspended by pushing the instruction pointer and frame pointer.
-  Once pushed, the `framePointer` can be overridden to mark the start of a new frame at the position of `stackPointer`.
- Then the method, `nil` for context, flags, receiver, and temps are pushed.




```caption=Setting up a frame&anchor=settingStackFrames
Interpreter >> setUpFrameForMethod: aMethod receiver: rcvr
	...
	numTemps := self temporaryCountOfMethodHeader: methodHeader.
	numArgs := self argumentCountOfMethodHeader: methodHeader.
	
	"Suspend current frame"
	self push: instructionPointer.
	self push: framePointer.

	"New frame starts"
	framePointer := stackPointer.
	self push: aMethod.
	self push: objectMemory nilObject.
	self push: (self
			 encodeFrameFieldHasContext: false
			 isBlock: false
			 numArgs: numArgs).
	self push: rcvr.

	"Initialize temps to nil"
	numArgs + 1 to: numTemps do: [ :i |
		self push: objectMemory nilObject ].
	...
```



### Bytecodes Accessing the Stack Frame

Given the structure of a stack, we can see that the all the fields in the fixed part of a frame can be found relative to the start of a frame, which for the first frame is the `framePointer`.
This includes in particular the receiver and the temporary variables.
Thus, the bytecode that access these values are defined to read/write at an offset from the frame pointer.

The code that follows shows the bytecode that pushes `self` to the stack:

```caption=The push receiver bytecode reads the receiver relative from the framePointer
Interpreter >> receiver
	^memory readWordAt: framePointer + FoxReceiver

Interpreter >> pushReceiverBytecode
	self fetchNextBytecode.
	self push: self receiver.
```

Moreover, to read/write the nth real temporary variable we need to read/write the nth _above_ the fixed fields of the frame.
The code below shows the bytecode that stores the top of the stack into a temporary variable and pops.
This bytecode uses the `itemporary:in:put:` method that is used to write into a method's temporary.
A similar method, `itemporary:in:` exists to read temporary variables.

Remember that the bytecode set is designed so arguments are treated as temporaries.
The code below shows that there are two execution paths: one for arguments, one for temporaries.
The interpreter decides if the asked offset is for a temporary of an argument by comparing it to the number of arguments.
The number of arguments is obtained from the fields flag.
In this section we focus on the path for temporaries.
We will explain the path for arguments in the next section.

The interpreter indexes temporary variables using as base the field that follows the receiver field,
and as offset the offset of the temporary without taking arguments into account.
In these two lines we see clearly that the stack grows down: to get the field _after_ we need to subtract from its position.
Moreover, since all accesses are written as direct memory addresses, all offsets are computed in bytes, thus multiplying by the number of bytes in a word `objectMemory wordSize`.


```caption=Storing
Interpreter >> iframeNumArgs: theFP
	^memory readByteAt: theFP + FoxFrameFlags + 1

Interpreter >> iframeReceiverLocation: theFP
	^theFP + FoxReceiver

Interpreter >> itemporary: offset in: theFP put: valueOop
	| frameNumArgs |
	
	^ offset < (frameNumArgs := self iframeNumArgs: theFP)
		  ifTrue: [ "Write an argument" ... ]
		  ifFalse: [
			  memory
				  writeWordAt:
					  (self iframeReceiverLocation: theFP) - objectMemory wordSize
					  + (frameNumArgs - offset * objectMemory wordSize)
				  put: valueOop ]
	

Interpreter >> storeAndPopTemporaryVariableBytecode
	self fetchNextBytecode.
	self
		itemporary: (currentBytecode bitAnd: 7)
		in: framePointer
		put: self stackTop.
	self pop: 1
```


### Conclusion

In this chapter we studied how the interpreter Pharo interpreter is written.
Both bytecode and primitive instructions are implemented through method dispatch.
The key instruction in Pharo is the message send.
When a message send instruction is executed, it first looks up the method to execute in the receiver's hierarchy.
Then, if it is a primitive method, it executes the associated primitive instruction.
If the primitive instruction fails or the method is a normal method, the method is _activated_: a frame is created and the method's bytecode gets executed.