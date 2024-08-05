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

The key operation in object oriented programs is the message send.
A message send is a late bound method activation on a receiver with a collection of arguments.
We say a message send is late bound because the decision of the method to execute is done at runtime, when the message send is executed.
Thus, message sends are implemented as two-step operations: first the interpreter finds the method to execute, then the method found is activated.

#### Preparing the Stack

Before executing a message send bytecode, all arguments, receiver included, need to be pushed to the stack in order.
That is, the receiver is pushed first, then each of the arguments from first to last, as illustrated in the following code.
The generated bytecode instructions must include the corresponding push instructions before the message send, as follows:

```caption=A send is preceeded by instructions that push the arguments
"Translation of 
  aNumber between: 17 and: 255"
push aNumber
push 17
push 255
send #between:and:
```

When a message send instruction is executed, the stack looks as follows:

![Preparing the Stack to send a message.](figures/interpreter_send.pdf?label=interpreterBeforeSend)


#### Message Send Bytecodes

To illustrate how message send bytecodes work, let's examine the source code of bytecode `extSendBytecode`.
The `extSendBytecode` bytecode is a two-byte bytecode implementing the general form of message sends.
The first byte is the opcode of the instruction, the second byte encodes the literal index where to find selector in the highest 5 bits, and the number of arguments of the selector in the lowest 3 bits.

After decoding the second byte and fetching the selector, the interpreter sets the interpreter variables `messageSelector` and `argumentCount`.
The argument count is used to fetch the receiver: the receiver is on the stack, above the arguments, so it can be obtained using the expression `self stackValue: argumentCount`.
Finally, we obtain the receiver's class index and set it to the interpreter variable `lkupClassTag`.
Both the `lkupClassTag` and `messageSelector` are used in the method lookup that is explained in the next section.

```caption=The general form send bytecode
Interpreter >> extSendBytecode
	| byte messageSelectorIndex messageArgumentCount |

	byte := self fetchByte.
	messageSelectorIndex := byte >> 3 + (extA << 5).
	messageArgumentCount := (byte bitAnd: 7) + (extB << 3).

	extA := 0.
	extB := 0.
	numExtB := 0.

	self
		normalLiteralSelectorAt: messageSelectorIndex
		argumentCount: messageArgumentCount

Interpreter >> normalLiteralSelectorAt: selectorIndex argumentCount: theArgumentCount

	messageSelector := self literal: selectorIndex.
	argumentCount := theArgumentCount.
	self normalSend

Interpreter >> normalSend

	| rcvr |
	rcvr := self stackValue: argumentCount.
	lkupClassTag := objectMemory fetchClassTagOf: rcvr.

	self commonSendOrdinary
```

#### Method Lookup Overview

The first step of interpreting a message send is finding the method to execute.
Starting from the class of the receiver, the method lookup algorithm linearly searches on each class in the hierarchy, as shown in the code below.
For each class, it takes the method dictionary from the second slot of the class object (`MethodDictionaryIndex` has value 1, second starting from 0), and looks up in the method dictionary for the method with the .
Here, the method `lookupMethodInDictionary:` has two possible outcomes: 
 - If the method is found, it sets the interpreter variable `newMethod` with the method found and returns `true`.
 - if the method is not found, it returns `false`.
If the method is found in the current method dictionary, the method `lookupMethodInClass:` returns

```caption=The lookup method
Interpreter >> findNewMethodInClassTag: classTagArg ifFound: aBlock
	"Entry was not found in the cache; look it up the hard way "
	lkupClass := objectMemory classForClassTag: classTag.
	self lookupMethodInClass: lkupClass.

Interpreter >> lookupMethodInClass: class
	| currentClass dictionary found |
	<inline: false>
	self assert: (self addressCouldBeClassObj: class).
	self lookupBreakFor: class.
	currentClass := class.
	[currentClass ~= objectMemory nilObject] whileTrue: [
		dictionary := objectMemory
			followObjField: MethodDictionaryIndex
			ofObject: currentClass.
		found := self lookupMethodInDictionary: dictionary.
		found ifTrue: [^currentClass].
		currentClass := self superclassOf: currentClass ].

	"We reached the top and found nothing..."
	...
```

#### Hashing and Structure of Method Dictionaries

Classes store method in a method dictionary.
A method dictionary is a hash table indexing methods by selector.
Method dictionaries are implemented using two arrays, an array of selector keys and an array of corresponding methods, associated by their index.
If a method dictionary has a given selector at index `n` of the selector array, then the corresponding method is at index `n` of the method array.

A note on the implementation is that method dictionaries do not actually employ two arrays.
The method dictionary itself _is_ the selector array.
For this, a method dictionary uses a hibrid layout with fixed instance variables and a variable indexable part used as the selector array.
The fixed instance variables are:
- `tally`: the number of occupied slots in the dictionary
- `array`: the method array

![Method dictionaries implement hash table using a double array.](figures/interpreter_method_dictionary.pdf?label=methoddict)

Although commonly method dictionaries use selector objects as keys, any object is accepted.
The search algorithm uses a simple hash function over the keys:
- a mask over the integer value of immediate keys.
- a mask over the selector's identity hash.
The mask used is the variable size of the method dictionary, ensuring that the index is always within bounds of the dictionary without extra checks.


```caption=The hashing function
Interpreter >> methodDictionaryHash: oop mask: mask
	^mask bitAnd: ((self isImmediate: oop)
		ifTrue: [self integerValueOf: oop]
		ifFalse: [self rawHashBitsOf: oop])

```

Small dictionaries, of size 8 or less by default, use a linear search and no hashing.
Larger dictionaries, instead, use a hash based lookup using open addressing with linear probing and wrap around.
That is, the hashed index is probed to check if it is either the searched element, an empty slot indicated by `nil`, or another element.
If the element is found or an empty slot is found, the search stops.
However, if another element is found in the hashed index, the next index if probed.
If the end of the array is reached, the search wraps and restarts from the beginning of the array.
If the original index is reached again after wrapping, the search stops.

```
Interpreter >> lookupMethodInDictionary: dictionary 
	| length index mask wrapAround nextSelector methodArray |

	length := objectMemory numSlotsOf: dictionary.
	mask := length - SelectorStart - 1.
	index := SelectorStart + (objectMemory methodDictionaryHash: messageSelector mask: mask).

	"Linear search on small dictionaries"
	mask <= methodDictLinearSearchLimit ifTrue:
		[index := 0.
		 [index <= mask] whileTrue:
			[nextSelector := objectMemory followField: index + SelectorStart ofObject: dictionary.
			 nextSelector = messageSelector ifTrue:
				[methodArray := objectMemory followObjField: MethodArrayIndex ofObject: dictionary.
				 newMethod := objectMemory followField: index ofObject: methodArray.
				^true].
		 index := index + 1].
		 ^false].

	"Hashed search with wrap around"
	index := SelectorStart + (objectMemory methodDictionaryHash: messageSelector mask: mask).
	wrapAround := false.
	[true] whileTrue:
		[nextSelector := objectMemory followField: index ofObject: dictionary.
		 nextSelector = objectMemory nilObject ifTrue: [^false].
		 nextSelector = messageSelector ifTrue:
			[methodArray := objectMemory followObjField: MethodArrayIndex ofObject: dictionary.
			 newMethod := objectMemory followField: index - SelectorStart ofObject: methodArray.
			^true].
		 index := index + 1.
		 index = length ifTrue:
			[wrapAround ifTrue: [^false].
			 wrapAround := true.
			 index := SelectorStart]].
````

#### Method Lookup and Super Sends

Super sends only differ from normal sends in the way the method lookup starts.
Instead of starting from the receiver's class, super sends start from the superclass of the current method's owner class, as shown in the expression `self superclassOf: (self methodClassOf: method).`
The method's owner class is stored in an association in last literal of the current method.

```
Interpreter >> extSendSuperBytecode
	| byte selectorIndex numArgs |

	byte := self fetchByte.
	selectorIndex := byte >> 3 + (extA << 5).
	numArgs := (byte bitAnd: 7) + (extB << 3).

	extA := 0.
	extB := 0.
	numExtB := 0.
	self sendSuper: selectorIndex numArgs: numArgs

Interpreter >> sendSuper: selectorIndex numArgs: numArgs

	messageSelector := self literal: selectorIndex.
	argumentCount := numArgs.
	self superclassSend

Interpreter >> superclassSend
	| superclass |

	superclass := self superclassOf: (self methodClassOf: method).
	lkupClassTag := objectMemory classTagForClass: superclass.
	self commonSendOrdinary

Interpreter >> methodClassOf: methodPointer
	| literal |

	literal := self
		followLiteral: (objectMemory literalCountOf: methodPointer) - 1
		ofMethod: methodPointer.
	^ (literal ~= objectMemory nilObject and: [objectMemory isPointers: literal])
		ifTrue: [ objectMemory followField: ValueIndex ofObject: literal ]
		ifFalse: [ objectMemory nilObject ]
```

#### Method Activation

#### The Stack Calling Convention

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

#### Does Not Understand

#### Cannot Interpret

### Interpreter optimizations

#### Where does the interpreter time go?

#### Global Lookup Cache

#### Static Type Predictions

#### Instruction Lookakeads

