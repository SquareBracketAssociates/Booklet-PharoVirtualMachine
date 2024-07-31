## The Interpreter

At the heart of the Pharo virtual machine there is the interpreter, Pharo's main execution engine.
Folklore says that interpreters are slow. Truth is they are. But they are also easy to implement and maintain.
They play a very important role in a multi-tiered architecture: optimized execution engines such as JIT compiler execute usually the commonly-executed code and fast execution paths, falling back to slower layers such as interpreters for the rest of the execution.
Thus, the Pharo interpreter implements the entire semantics of Pharo: all bytecodes and primitives.
This makes it a good target to study and understand how Pharo works.

This chapter visits the interpreter from a high-level point of view, using code excerpts and explaining along the way some basics of how interpreters work.
This includes how the interpreter manages its execution state and the call stack, how it dispatches instructions, some illustrative instructions and some of its platform independent optimizations: instruction lookahead and static type predictions.
The chapter on Slang VM generation presents machine dependent optimizations such as interpreter threading and autolocalization.

A section of this chapter is dedicated to message sends: the most important instruction in the Pharo programming language and in many object oriented languages.
Message-sends, as similar as they may look to statically-bound function calls, must implement late-bound execution.
Sending a message requires looking up methods up in the receiver class hierarchy, which is one of the most common operations in Pharo, and one of the most complex and expensive too.
One simple way to cope with such costs are global lookup caches.


### A Bytecode Interpreter

Pharo's execution engine is based on the execution of bytecode instructions.
Bytecode instructions are organized as a list of instructions, where one instruction is executed after the other.
For each instruction, an interpreter typically follows two main steps: fetching the next instruction and dispatching to the code implementing the instruction.
Typically, interpreters are implemented using a `while` and a `switch` statement.
The `while` statement executes indefinitely (or until the program exits).
The `switch` statement contains a case per instruction and dispatches to the corresponding code depending on the read instruction.
The following code illustrates a simple interpreter written a the C programming language.

```caption=Sketching an interpreter in C
void interpret(){
	while(true){
		int instruction = fetchNextInstruction();
		switch(instruction){
			case INSTRUCTION_1:
			   ...
			   break;
			case INSTRUCTION_2:
			   ...
			   break;
			...
		}
	}
}
```

#### Table dispatch

Pharo's bytecode interpreter is not written as a case statement as shown before.
Instead, it uses a table dispatch mechanism, where a table, namely the _bytecode table_ associates an instruction with the code to execute for that instruction.
The bytecode table is an array ranging from 0 to 255, thus covering the entire bytecode set, and containing unary selectors.
Instructions are dispatched by looking up the selector for an instruction, then using the selector to send a message to the interpreter using `#perform:`.

```caption=Pharo interpreter uses a table dispatch with symbols
Interpreter >> interpret
	...
	self fetchNextBytecode.
	[ true ] whileTrue: [
		self dispatchOn: currentBytecode in: BytecodeTable
	].
	...

Interpreter >> dispatchOn: anInteger in: selectorArray

	self perform: (selectorArray at: (anInteger + 1)).
````

#### Instruction Fetching and the Instruction Pointer

Mimicking how the CPU handles its own program counter, the Pharo interpreter manages its instruction stream using a variable called `ìnstructionPointer`, which is a pointer to the current instruction in the method under execution.
Using the instruction pointer, the interpreter fetches instructions one-by-one sequentially from a method's bytecode list.

```caption=Low-level stack operations in the interpreter
Interpreter >> fetchNextBytecode
	^objectMemory byteAt: (instructionPointer := instructionPointer + 1)
```

The instruction pointer is not only manipulated to fetch instructions.
Later in this chapter we will see how bytecode instructions manipulate it to implement control flow operations: jumps, message sends and returns.
Moreover, later chapters will show the role of the instruction pointer to implement green-thread based concurrency.

**Each bytecode instruction fetches the next instruction.** We will observe in the methods defining bytecodes that, differently from the interpreter C sketch at the beginning of the chapter, each instruction is responsible for fetching its next instruction. This implementation detail duplicates the instruction fetching code around the interpreter, while simplifying its translation to a threaded interpreter as it will be explained in the Slang chapter.

#### Pushing and Popping, the Stack and the Stack Pointer

The execution of Pharo code is supported mainly by a stack supporting operations such as push, pop and indexed access from the top.
The stack is a contiguous region of memory of fixed size organized in slots of one word each.
The _base_ of the stack is where the stack begins, where its oldest elements are stored.
Conversely, the _top_ of the stack is where the most recently pushed element is stored.
The stack is manipulated through a variable called `stackPointer`, which is a pointer to the top of the stack.

**The stack grows down:** in Pharo, as in many other language implementations, the stack base is in a higher address than the stack top.
Pushing a value to the stack moves the stack pointer towards lower addresses. Popping moves it towards higher addresses.

Using the `stackPointer` variable the interpreter implements the following methods:

```caption=Low-level stack operations in the interpreter
Interpreter >> stackValue: offset
	^memory readWordAt: stackPointer + (offset*objectMemory wordSize)

Interpreter >> stackTop
	^ self stackValue: 0

Interpreter >> push: object
	memory readWordAt: (sp := stackPointer - objectMemory wordSize) put: object.
	stackPointer := sp

Interpreter >> pop: nItems
	stackPointer := stackPointer + (nItems*objectMemory wordSize).
	^nil

Interpreter >> popStack
	| top |
	top := self stackTop.
	self pop: 1.
	^top
```

Given these basic operations, we can already understand several simple instructions such pushing well-known constants, popping from the stack, or duplicate top.

```caption=Bytecode implementing simple push and pop instructions
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

### The Call Stack

Many bytecode instructions require accessing data that belongs to a particular method execution: the receiver, arguments, temporaries or the current method to obtain its literals. Such information is stored in the stack.

The stack is split in regions called _stack frames_, or frames for short.
The frames in the stack form a chain of frames, usually referred to as the _call stack_.
In the top of the stack there is the top stack frame representing the current method execution.

#### Stack Frame Structure

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

![The structure of a stack frame and the interpreter pointers.](figures/interpreter_variables.pdf?label=interpreterVariables)

The frame on the top of the stack is said to be _active_.
The active frame is delimited by the variables `stackPointer` at its top and `framePointer` at its base.
While the `stackPointer` will move with each push/pop instruction, the `framePointer` points to the base of the current frame in the stack and does not change during a method execution.
Moreover, the `instructionPointer` is a pointer within the bounds of the method of the active frame.

All frames other than the active one are said to be _suspended_.
Suspended frames need the values of their `framePointer`, `stackPointer` and `instructionPointer` to be stored so _e.g.,_ the stack can be traversed by the garbage collector and control can return to those frames after a return instruction.
A frame is suspended by pushing its instruction pointer to the stack before creating a new frame, and pushing its frame pointer as the first element of the next frame. Storing on each frame the frame pointer of the preceding frame transforms the stack in a linked lists of frames.
Thus, the stack can be reconstructed by iterating from the top frame up to its caller's frame start until the end of the stack.
Notice that the stack pointer needs not to be stored: a suspended frame's stack pointer is the slot that precedes its suspended instruction pointer, which is found relative to its following frame.

![The call stack as a linked lists of frames in the stack.](figures/interpreter_call_stack.pdf?label=interpreter_stack)


#### Stack Frame Flags

Within each interpreter stack frame there is the flags, a bitfield storing interpreter meta-data about the frame.
The two figures below show the structure of the flags in 64 and 32 bits architectures.
The stack frame flags store, from lower to higher bits in the bit field:
- **Method number of arguments:** A 1-byte field _caching_ the number of arguments, to avoid fetching it from the method.
- **hasContext flag:** A 1-byte field used as a boolean, indicating if the frame has been reified or not.
- **isBlock flag:** Only available in non-JIT VMs: It's a 1-byte field used as a boolean, indicating if the frame is for a block closure execution or not. In JIT VMs, this flag is available as as a tag in the method pointer.
- **Number of backjumps performed in this method:** Only available in JIT VMs. It's a 1-byte field used as a counter, indicating how many backjumps where performed by this method execution during interpretation. If this field goes over a threshold, meaning that a long loop is being interpreted, the JIT compiler decides to compile the method and switch to compiled execution.

![Structure of the frame flags in 64 bits.](figures/interpreter_flags.pdf?label=interpreterFlags)

Notice that the frame flags have a first constant field in the range 0-7 with value 1.
This value is set to make the bitfield look as a tagged small integer, thus guarding the GC walking the stack from interpreting this value as a pointer.

#### Setting up Stack Frames

The code that follows illustrates how a new frame is created.
First the current frame is suspended by pushing the instruction pointer and frame pointer.
Once pushed, the `framePointer` can be overridden to mark the start of a new frame at the position of `stackPointer`.
Then the method, `nil` for context, flags, receiver and temps are pushed.

```caption=Setting up a frame
Interpreter >> setUpFrameForMethod: aMethod receiver: rcvr

	| methodHeader numArgs numTemps rcvr |
	methodHeader := objectMemory methodHeaderOf: aMethod.
	numTemps := self temporaryCountOfMethodHeader: methodHeader.
	numArgs := self argumentCountOfMethodHeader: methodHeader.
	
	"Suspend current frame"
	self push: instructionPointer.
	self push: framePointer.

	"New frame starts"
	framePointer := stackPointer.
	self push: newMethod.
	self push: objectMemory nilObject. "FxThisContext field"
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

#### Bytecodes Accessing the Stack Frame

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

### Interpreting Message sends

#### Calling Convention

The interpreter and the JIT share the same calling conventions. The receiver and arguments are pushed on the stack. The instruction pointer is also pushed to the stack after the arguments. 

Figure *@stackGrowing@* shows that the stack is growing down from high addresses to low addresses.
This convention is important and has an impact of many aspect such as object allocation and different logic in garbage collector implementation.

![Stack growing down.](figures/StackGrowingDown.pdf width=30&label=stackGrowing)


Figure *@beforesend@* shows that the before doing a call, the receiver and arguments are pushed to the stack.

![Receiver and arguments are pushed to the stack.](figures/BeforeSend.pdf width=80&label=beforesend)

Figure *@aftersend@* shows that the instruction pointer (IP) is also pushed to the stack. This way it is possible to find which instruction is the next one to execute on return. Notice also that the interpreter and the VM are __caller-saved__. It means that this is the caller responsibility to store information that should be recovered on return of the called function.

![Caller saved: the IP is also pushed to make sure that the caller can know the next instruction on return.](figures/AfterSend.pdf width=80&label=aftersend)

Figure *@generalArguments@* shows that the framepointer is used to compute 
- method argument. Since the arguments are pushed on the stack before the new frame is allocated, a method argument is always computed as an addition to the framepointer (`arg1 = FP + arg1offset`).
- method. The method (with its metadata) is located at a fix offset from the frame pointer. Hence `method= FP- method offset`.


![arg1 = FP + offset and method = FP - method offset.](figures/GeneralArgument.pdf width=100&label=generalArguments)


#### Method lookup

#### Method Activation

### Interpreter optimizations

#### Where does the interpreter time go?

#### Global Lookup Cache

#### Static Type Predictions

#### Instruction Lookakeads

