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


>[! Important] 
> The stack grows down: in Pharo, as in many other language implementations, the stack base is in a higher address than the stack top.
Pushing a value to the stack moves the stack pointer towards lower addresses. Popping moves it towards higher addresses.


![The stack grows down. %width=70&anchor=interpreterVariables](figures/interpreter_variables.pdf)


Using the `stackPointer` variable the interpreter implements the following methods shown in Listing *@stackOps@*.

```caption=Low-level stack operations in the interpreter&anchor=stackOps
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

![The structure of a stack frame and the interpreter pointers. %width=70&anchor=interpreterVariables](figures/interpreter_variables.pdf)

The frame on the top of the stack is said to be _active_.
The active frame is delimited by the variables `stackPointer` at its top and `framePointer` at its base.
While the `stackPointer` will move with each push/pop instruction, the `framePointer` points to the base of the current frame in the stack and does not change during a method execution.
Moreover, the `instructionPointer` is a pointer within the bounds of the method of the active frame.

All frames other than the active one are said to be _suspended_.
Suspended frames need the values of their `framePointer`, `stackPointer` and `instructionPointer` to be stored so _e.g.,_ the stack can be traversed by the garbage collector and control can return to those frames after a return instruction.
A frame is suspended by pushing its instruction pointer to the stack before creating a new frame, and pushing its frame pointer as the first element of the next frame. Storing on each frame the frame pointer of the preceding frame transforms the stack in a linked lists of frames.
Thus, the stack can be reconstructed by iterating from the top frame up to its caller's frame start until the end of the stack.
Notice that the stack pointer needs not to be stored: a suspended frame's stack pointer is the slot that precedes its suspended instruction pointer, which is found relative to its following frame.

![The call stack as a linked lists of frames in the stack. %width=70&anchor=interpreter_stack](figures/interpreter_call_stack.pdf)


### Setting up Stack Frames

Listing *@settingStackFrames@* shows how a new frame is created.

- First the current frame is suspended by pushing the instruction pointer and frame pointer.
-  Once pushed, the `framePointer` can be overridden to mark the start of a new frame at the position of `stackPointer`.
- Then the method, `nil` for context, flags, receiver, and temps are pushed.

```caption=Setting up a frame&anchor=settingStackFrames
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

The key operation in object-oriented programs is the message send.
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

![Preparing the Stack to send a message. %width=60&anchor=interpreterBeforeSend](figures/interpreter_send.pdf)


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

### Method Lookup Overview

The first step of interpreting a message send is finding the method to execute.
Starting from the class of the receiver, the method lookup algorithm linearly searches on each class in the hierarchy, as shown in the code below (Listing *@methodlookup@*).

For each class, it takes the method dictionary from the second slot of the class object (`MethodDictionaryIndex` has value 1, second starting from 0), and looks up in the method dictionary for the method with the ???.

Here, the method `lookupMethodInDictionary:` has two possible outcomes: 

- If the method is found, it sets the interpreter variable `newMethod` with the method found and returns `true`.
- if the method is not found, it returns `false`.

If the method is found in the current method dictionary, the method `lookupMethodInClass:` returns.

```caption=The lookup method&anchor=methodlookup
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


### Hashing and Structure of Method Dictionaries

Classes store methods in a method dictionary.
A method dictionary is a hash table indexing methods by selector.
Method dictionaries are implemented using two arrays, an array of selector keys, and an array of corresponding methods, associated by their index.
If a method dictionary has a given selector at index `n` of the selector array, then the corresponding method is at index `n` of the method array.

A note on the implementation is that method dictionaries do not actually employ two arrays.
The method dictionary itself _is_ the selector array.
For this, a method dictionary uses a hybrid layout with fixed instance variables and a variable indexable part used as the selector array (See Figure *@methoddict@*).
The fixed instance variables are:

- `tally`: the number of occupied slots in the dictionary
- `array`: the method array

![Method dictionaries implement hash table using a double array. %width=60&anchor=methoddict](figures/interpreter_method_dictionary.pdf)

Although method dictionaries use selector objects as keys, any object is accepted.
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
Larger dictionaries, instead, use a hash-based lookup using open addressing with linear probing and wrap around.
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

### Method Lookup and Super Sends

Super sends only differ from normal sends in the way the method lookup starts.
Instead of starting from the receiver's class, super sends start from the superclass of the current method's owner class, as shown in the expression `self superclassOf: (self methodClassOf: method).`
The method's owner class is stored in an association in the last literal of the current method.

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

### Method Activation

Once the method to execute is found, the following step is to activate the method in the stack.
Activating a method implies creating a stack frame for it and prepare the interpreter to execute the instructions in that method.
Frames are created by pushing the current instruction pointer, thus suspending the current frame, and then pushing all frame fields, as explained earlier in this chapter.

```
Interpreter >> activateNewMethod
    
	"set up the stack frame for the method"
	...
	
	"Set the interpreter instruction pointer to the first instruction of the method"
    instructionPointer := self pointerForOop:
		                      (self
			                       initialIPForHeader: methodHeader
			                       method: newMethod) - 1.
```

### Handling Return instructions

Related to send instructions are return instructions which return the control to the caller method.
A return instruction has to _undo_ what a send instruction does.
The current stack frame needs to be destroyed, and the instruction pointer, frame pointer and stack pointer of the caller frame must be restored.
All of the caller frame's state is accessible relative to the current's frame base.
Notice that we do not need to clean the stack and moving the stack pointer upwards is enough.
All values after the stack pointer become unreachable, and will eventually get overwritten on a next message send.

In addition, remember that return instructions have a return value that need to be given to the caller.
Passing the return value to the caller is made through the stack.
It's the callee's responsibility to pop the receiver and arguments and push the return value upon return.

```caption=Returning from a method pops restores the caller's stack frame, pops the message receiver and arguments, and pushes the return value
Interpreter >> returnTopFromMethod

	self commonReturn: self popStack

Interpreter >> commonReturn: value

	localReturnValue := value.
	self commonCallerReturn

Interpreter >> commonCallerReturn
	| callersFPOrNull |

	callersFPOrNull := self frameCallerFP: framePointer.

	instructionPointer := self frameCallerSavedIP: framePointer.
	stackPointer := framePointer
	                + (self frameStackedReceiverOffset: framePointer).
	framePointer := callersFPOrNull.

	self setMethod: (self iframeMethod: framePointer).
	self fetchNextBytecode.
	self stackTopPut: localReturnValue

Interpreter >> frameStackedReceiverOffset: theFP

	^self frameStackedReceiverOffsetNumArgs: (self frameNumArgs: theFP)
```

The following figure shows the stack after a return instruction: the three interpreter variables have been restored destroying the callee frame.
In the caller's frame, the stack values representing the receiver and arguments have been popped, the return value has been pushed.

![The stack before and after a return instruction.%anchor=return](figures/interpreter_return.pdf)

#### Calling Convention Summary

Now that we have seen in detail how message sends and the call stack work, let us summarize the calling convention:

- Receiver and arguments are pushed to the stack by the caller before the send instruction
- Receiver and arguments remain on the stack, and are accessed relative to the new frame's base
- Upon return, it's the callee's responsibility to pop the receiver and arguments and to push the return value to the stack

We will slightly revisit this calling convention when introducing just-in-time compilation.


### Interpreting Primitives

So far we have concentrated on interpreting bytecodes.
In this  section we concentrate on the interpretation of primitive methods.

Recall that primitive methods are normal methods that contain a reference to a primitive function.
For example, the next code snippet shows the method `SmallInteger>>+` implementing the addition of two tagged integers.
This method has the pragma `primitive: 1` indicating that this method is a primitive method, referencing the primitive 1.

```caption=The primitive method implementing addition in SmallInteger
SmallInteger >> + aNumber

	<primitive: 1>
	^ super + aNumber
````

#### Primitive Instructions and Calling Convention by Example

Primitive instructions are stack-based instructions with a catch: they can fail.

When called, primitive instructions expect all its arguments on the stack, plus the interpreter variable `numberOfArguments` set to the actual number of arguments of the message send.
Thus, primitive instructions have two return values: a failure/success code, and an optional return value if success.
To allow this, primitive instructions store their error code in an interpreter variable, and their result on the stack.
Invoking a primitive instruction follows the convention on return:
 - set the primitive failure code variable `primFailCode` to 0 on success, or an error code otherwise
 - if failure, leave the stack untouched
 - if success, pop the arguments, then push the result

The following code illustrates such concepts with `primitiveAdd` the primitive instruction that adds two tagged small integers.
`primitiveAdd` first fetches it's two arguments from the top two slots of the stack.
Second, it checks if they are integer objects: if they are not, it returns indicating a failure.
Third, it sums both integer values, and in case of overflow (_i.e.,_ in case the addition cannot be represented as a tagged small integer) it returns indicating a failure.
Finally, if the sum does not overflow, the two arguments are popped from the stack and the result is pushed.

```
Interpreter >> primitiveAdd	
	| maybeSmallInteger maybeSmallInteger2 result |

	maybeSmallInteger := self stackValue: 0.
	maybeSmallInteger2 := self stackValue: 1.
	
	(objectMemory isIntegerObject: maybeSmallInteger)
		ifFalse: [ ^ self primitiveFail ].
	(objectMemory isIntegerObject: maybeSmallInteger2)
		ifFalse: [ ^ self primitiveFail ].

	"Check for overflow"
	result := self
		sumSmallInteger: maybeSmallInteger
		withSmallInteger: maybeSmallInteger2
		ifOverflow: [ ^ self primitiveFail ].

	self pop: 2 thenPush: result
```

Notice how this primitive does not do any destructive operation on the stack unless we are sure that it cannot fail.
Alternatively, if it popped the top elements of the stack at the beginning, the stack should have been rebuilt upon failure, by re-pushing the arguments in the right order. 

**Design Note: when should a primitive fail?**
The design of primitives is revolved around type-safety and memory-safety.
Since Pharo is a dynamically-typed programming language, types are not know in advance.
Instead, types should be checked at runtime before performing dangerous operations.
In the example above it checks the types of the arguments _and_ the non-overflow of the addition.
Primitives that load/store from/into object slots type check the receiver and index arguments, but also check that the index is within bounds.
Notice that such a design guide primitives to provide a clear separation between the _fast and slow paths_.
Primitives generally provide a safe implementation for a fast path, leaving the slow path to the fallback code.

### Decoding a Method's Primitive Index

We distinguish primitive methods from non-primitive methods using a bit in their first literal.
The primitive index of a primitive method is stored in its first bytecode instruction: a call primitive bytecode.
The call primitive bytecode is a 3-byte bytecode instruction with opcode 248.
The two subsequent bytes represent the primitive index in little endian: `byte1 + (byte2 << 8)`.
The following code shows how the runtime extracts the method primitive index of a method.

```
Interpreter >> primitiveIndexOf: methodPointer
	^self primitiveIndexOfMethod: methodPointer header: (objectMemory methodHeaderOf: methodPointer)

Interpreter >> primitiveIndexOfMethod: theMethod header: methodHeader
	| firstBytecode |
	firstBytecode := self
		firstBytecodeOfHeader: methodHeader
		method: theMethod.
	^ (objectMemory byteAt: firstBytecode + 1)
		+ ((objectMemory byteAt: firstBytecode + 2) << 8)

Interpreter >> firstBytecodeOfHeader: methodHeader method: theMethod
	^theMethod
	 + ((LiteralStart + (self literalCountOfHeader: methodHeader)) * objectMemory bytesPerOop)
	 + objectMemory baseHeaderSize

Interpreter >> literalCountOfHeader: headerPointer
	^ (objectMemory integerValueOf: headerPointer)
		bitAnd: HeaderNumLiteralsMask
````

### Primitive Dispatch and Failure Fallback

When a method is executed, the interpreter checks first if the method is a primitive method _i.e.,_ if it has its flag turned on.
If that is the case, the interpreter invokes the primitive instead of activating the method.
As with bytecode instructions, primitives use also table dispatch to map primitive numers (or indexes) to primitive functions.
The code that follows shows a sketch of the primitive table.

```
	#(	(1 primitiveAdd)
		(2 primitiveSubtract)
		(3 primitiveLessThan)
		(4 primitiveGreaterThan)
		(5 primitiveLessOrEqual)
		(6 primitiveGreaterOrEqual)
		(7 primitiveEqual)
		(8 primitiveNotEqual)
		(9 primitiveMultiply)
		(10 primitiveDivide)
		(11 primitiveMod)
		...
	)
```

To execute the primitive, we fetch the primitive function from the table, then we dispatch the primitive using `perform:`.
The following piece of code shows a simplified version of Pharo's interpreter code.
First, the interpreter decodes the primitive index and fetches the primitive function pointer.
If the method has an associated primitive function, the function is executed.
If success, the method returns directly to the caller.
Otherwise, the method is activated, forcing the execution of the fallback code in the method's bytecode.

```
primitiveIndex := self primitiveIndexOf: newMethod.
primitiveFunctionPointer := primitiveTable at: primitiveIndex.
primitiveFunctionPointer ~= 0 ifTrue: [ | success |
	self perform: primitiveFunctionPointer.

	self success ifTrue: [
		"If success, return to the caller"
		^ nil
	]
].

"handle primitive error or no primitive"
self activateNewMethod
```

### When the Lookup Fails: VM Callbacks

So far we assumed that the message lookup does always find a method to execute, although this may not always be the case.
In this section we explain two interpreter mechanisms that handle lookup errors: does not understand and cannot interpret.
From an interpreter point of view, these mechanisms are _callbacks_ to Pharo code indicating the problematic cases.

#### Does Not Understand

When a message is sent, the method lookup searches the entire receiver's hierarchy for a method with the same selector as the message.
If the algorithm reaches the top of the hierarchy (`nil`), then the interpreter sends the `doesNotUnderstand:` message to the receiver.
The interpreter massages a bit the stack and the interpreter state, and looks up the `doesNotUnderstand:` selector instead of the original selector.
This creates the illusion that the VM replaced the original message by a `doesNotUnderstand:` message.

Massaging the stack implies preparing it to send the `doesNotUnderstand:` message.
The thing is, the stack contains the receiver and arguments of the message but `doesNotUnderstand:` expects as single argument: the reification of the original message.
Then the original message reification is set up by:
- allocating an array to hold all the original arguments
- popping the arguments from the stack and store them in the new array
- allocating a message object and storing in it the original selector, argument array and lookup class

![Reifying the message for does not understand. %width=80&anchor=dnu_message](figures/interpreter_dnu.pdf)

Then, the `messageSelector` interpreter variable is overwritten with the `doesNotUnderstand:` selector found in the special objects array, and the lookup is restarted.

```caption=The lookup method revisited with `doesNotUnderstand:` support
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

	"Cound not find a normal message -- raise exception #doesNotUnderstand:"
	self createActualMessageTo: class.
	messageSelector := objectMemory splObj: SelectorDoesNotUnderstand.
	^self lookupMethodInClass: class
```

### Cannot Interpret

The VM implements also the `cannotInterpret:` hook: a hook that gets activated when the VM finds `nil` in the place of a class' method dictionary.
Similar to `doesNotUnderstand:`, `cannotInterpret:` requires massaging the stack and setting up the message reification.
However, the lookup cannot be restarted from the original class, as this will find again a `nil` method dictionary and _loop_.
Instead, `cannotInterpret:` restart's the lookup from the superclass of the class that had a `nil` method dictionary, cutting up the recursion.

```caption=The lookup method
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
		
		dictionary = objectMemory nilObject ifTrue:
			["MethodDict pointer is nil (hopefully due a swapped out stub)
				-- raise exception #cannotInterpret:."
			self createActualMessageTo: class.
			messageSelector := objectMemory splObj: SelectorCannotInterpret.
			^self lookupMethodInClass: (self superclassOf: currentClass)].
		
		found := self lookupMethodInDictionary: dictionary.
		found ifTrue: [^currentClass].
		currentClass := self superclassOf: currentClass ].
	...
```

### Conclusion

In this chapter we studied how the interpreter Pharo interpreter is written.
Both bytecode and primitive instructions are implemented through method dispatch.
The key instruction in Pharo is the message send.
When a message send instruction is executed, it first looks up the method to execute in the receiver's hierarchy.
Then, if it is a primitive method, it executes the associated primitive instruction.
If the primitive instruction fails or the method is a normal method, the method is _activated_: a frame is created and the method's bytecode gets executed.