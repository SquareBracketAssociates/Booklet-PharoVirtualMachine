## The Interpreter
@cha:interpreter

At the heart of the Pharo virtual machine there is the interpreter, Pharo's main execution engine.
Folklore says that interpreters are slow. Truth is they are. But they are also easy to implement and maintain.
They play a very important role in a multi-tiered architecture: optimized execution engines (such as JIT compilers) execute usually the commonly-executed code and fast execution paths, falling back to slower layers such as interpreters for the rest of the execution.
Thus, the Pharo interpreter implements the entire semantics of Pharo: all bytecodes and primitives.
This makes it a good target to study and understand how Pharo works.

This chapter visits the interpreter from a high-level point of view, using code excerpts and explaining along the way some basics of how interpreters work.
This includes how the interpreter manages its execution state and the call stack, how it dispatches instructions, some illustrative instructions and some of its platform independent optimizations: instruction lookahead and static type predictions.
The chapter on Slang VM generation presents machine dependent optimizations such as interpreter threading and autolocalization.

A section of this chapter is dedicated to message sends: the most important instruction in the Pharo programming language and in many object-oriented languages.
Message sends, as similar as they may look to statically-bound function calls, must implement late-bound execution.
Sending a message requires looking up the corresponding method in the receiver's class hierarchy, which is one of the most common operations in Pharo, and one of the most complex and expensive too.
One simple way to cope with such costs are global lookup caches.


### A Bytecode Interpreter

Pharo's execution engine is based on the execution of bytecode instructions.
Bytecode instructions are organized as a list of instructions, where one instruction is executed after the other.
An interpreter typically follows two main steps for each instruction: it fetches the next instruction and dispatches to the code that implements the instruction.

Typically, interpreters are implemented using a `while` and a `switch` statement.
The `while` statement executes indefinitely (or until the program exits).
The `switch` statement contains a case per instruction and dispatches to the corresponding code depending on the instruction.
The code in Listing *@Cinterp@* illustrates a simple interpreter written the C programming language.

```caption=Sketching an interpreter in C.&anchor=Cinterp
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
Instructions are dispatched by looking up the selector for an instruction, then using the selector to send a message to the interpreter using the message `perform:`, as shown in Listing *@dispatch@*.

Note that while we write the message `perform:`, the C transpiler will generate a call to the corresponding function implementing the bytecode semantics.


```caption=Pharo interpreter uses a table dispatch with symbols.&anchor=dispatch
Interpreter >> interpret
	...
	self fetchNextBytecode.
	[ true ] whileTrue: [
		self dispatchOn: currentBytecode in: BytecodeTable
	].
	...

Interpreter >> dispatchOn: anInteger in: selectorArray
	self perform: (selectorArray at: (anInteger + 1))
````

### Instruction Fetching and the Instruction Pointer

Mimicking how the CPU handles its own program counter, the Pharo interpreter manages its instruction stream using a variable called `ìnstructionPointer`, which is a pointer to the current instruction in the method under execution.

Using the instruction pointer, the interpreter fetches instructions one by one sequentially from a method's bytecode list as shown in Listing *@fetchnext@*. Here the variable `objectMemory` is a pointer to the complete memory of the interpreter and the instructionPointer is just an address within the range of the currently executed compiled method.

```caption=Low-level stack operations in the interpreter.&anchor=fetchnext
Interpreter >> fetchNextBytecode
	currentByteCode := objectMemory byteAt: (instructionPointer := instructionPointer + 1)
```

>[! note ]
> Each bytecode instruction fetches the next instruction. We will observe in the methods defining bytecodes that, differently from the interpreter C sketch at the beginning of the chapter, each instruction is responsible for fetching its next instruction. This implementation detail duplicates the instruction fetching code around the interpreter, while simplifying its translation to a threaded interpreter as it will be explained in the Slang chapter.

The instruction pointer is not only manipulated to fetch instructions.
Later in this chapter, we will see how bytecode instructions manipulate it to implement control flow operations: jumps, message sends, and returns.
Moreover, later chapters will show the role of the instruction pointer to implement green-thread based concurrency.

### Interpreting Message sends

The key operation in object-oriented programs is the message send.
A message send is a late bound method activation on a receiver with a collection of arguments.
We say a message send is late bound because the decision of the method to execute is done at runtime, when the message send is executed.
Thus, message sends are implemented as two-step operations: first the interpreter finds the method to execute, then the method found is activated.

#### Preparing the Stack

Before executing a message send bytecode, all arguments, receiver included, have to be pushed to the stack in order.
That is, the receiver is pushed first, then each of the arguments from first to last, as illustrated in the following code.
The generated bytecode instructions must include the corresponding push instructions before the message send, as follows:

```caption=A send is preceded by instructions that push the arguments.
"Translation of aNumber between: 17 and: 255"
push aNumber
push 17
push 255
send #between:and:
```

When a message send instruction is executed, the stack looks as follows:

![Preparing the stack to send a message. %width=60&anchor=interpreterBeforeSend](figures/interpreter_send.pdf)


### Message Send Bytecodes

To illustrate how message send bytecodes work, let's examine the source code of bytecode `extSendBytecode`, as shown in Listing *@extSendBytecode@*.
The `extSendBytecode` bytecode is a two-byte bytecode implementing the general form of message sends.
The first byte is the opcode of the instruction, the second byte encodes the literal index where to find the selector in the highest 5 bits, and the number of arguments of the selector in the lowest 3 bits.

After decoding the second byte and fetching the selector, the interpreter sets the interpreter variables `messageSelector` and `argumentCount`.
The argument count is used to fetch the receiver: the receiver is on the stack, above the arguments, so it can be obtained using the expression `self stackValue: argumentCount`.
Finally, we obtain the receiver's class index and set it to the interpreter variable `lkupClassTag`.
Both the `lkupClassTag` and `messageSelector` are used in the method lookup that is explained in the next section.

```caption=The general form send bytecode.&anchor=extSendBytecode
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
Starting from the class of the receiver, the method lookup algorithm linearly searches on each class in the hierarchy, as shown in the Listing *@methodlookup@*.

For each class, it takes the method dictionary from the second slot of the class object (`MethodDictionaryIndex` has value 1, second starting from 0), and looks up in the method dictionary for the method with the ???.

Here, the method `lookupMethodInDictionary:` has two possible outcomes:

- If the method is found, it sets the interpreter variable `newMethod` with the method found and returns `true`.
- if the method is not found, it returns `false`.

If the method is found in the current method dictionary, the method `lookupMethodInClass:` returns.

```caption=The lookup method.&anchor=methodlookup
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

![Method dictionaries implement a hash table using a double array. %width=60&anchor=methoddict](figures/interpreter_method_dictionary.pdf)

Although method dictionaries use selector objects as keys, any object is accepted.
The search algorithm uses a simple hash function over the keys:

- a mask over the integer value of immediate keys.
- a mask over the selector's identity hash.

The mask used is the variable size of the method dictionary, ensuring that the index is always within bounds of the dictionary without extra checks.


```caption=The hashing function.
Interpreter >> methodDictionaryHash: oop mask: mask
	^mask bitAnd: ((self isImmediate: oop)
		ifTrue: [self integerValueOf: oop]
		ifFalse: [self rawHashBitsOf: oop])

```

- Small dictionaries, of size 8 or less by default, use a linear search and no hashing.
- Larger dictionaries, instead, use a hash-based lookup using open addressing with linear probing and wrap around. That is, the hashed index is probed to check if it is either the searched element, an empty slot indicated by `nil`, or another element.
If the element is found or an empty slot is found, the search stops. However, if another element is found in the hashed index, the next index if probed. If the end of the array is reached, the search wraps and restarts from the beginning of the array. If the original index is reached again after wrapping, the search stops.

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
Frames are created by pushing the current instruction pointer, thus suspending the current frame, and then pushing all frame fields, as explained earlier in Chapter *@cha:stackframe@*.

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
Passing the return value to the caller is done through the stack.
It's the callee's responsibility to pop the receiver and arguments and push the return value upon return.

```caption=Returning from a method restores the caller's stack frame, pops the message receiver and arguments, and pushes the return value.
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

Figure *@return@* shows the stack after a return instruction: the three interpreter variables have been restored destroying the callee frame.
In the caller's frame, the stack values representing the receiver and arguments have been popped, and the return value has been pushed.

![The stack before and after a return instruction.%anchor=return](figures/interpreter_return.pdf)

#### Calling Convention Summary

Now that we have seen in detail how message sends and the call stack work, let us summarize the calling convention:

- Receiver and arguments are pushed to the stack by the caller before the send instruction
- Receiver and arguments remain on the stack, and are accessed relative to the new frame's base
- Upon return, it's the callee's responsibility to pop the receiver and arguments and to push the return value on the stack.

We will slightly revisit this calling convention when introducing just-in-time compilation.


### Interpreting Primitives

So far we have concentrated on interpreting bytecodes.
In this  section we concentrate on the interpretation of primitive methods.

Recall that primitive methods are normal methods that contain a reference to a primitive function.
For example, the code snippet in Listing *@addition@* shows the method `SmallInteger>>+` implementing the addition of two tagged integers.
This method has the pragma `primitive: 1` indicating that this method is a primitive method, referencing the primitive 1.

```caption=The primitive method implementing addition in SmallInteger.&anchor=addition
SmallInteger >> + aNumber

	<primitive: 1>
	^ super + aNumber
````

#### Primitive Instructions and Calling Convention by Example

Primitive instructions are stack-based instructions with a catch: they can fail, contrary to bytecode which cannot.

When called, primitive instructions expect all its arguments on the stack, plus the interpreter variable `numberOfArguments` set to the actual number of arguments of the message send.
Thus, primitive instructions have two return values: a failure/success code, and an optional return value if success.
To allow this, primitive instructions store their error code in an interpreter variable, and their result on the stack.
Invoking a primitive instruction follows the convention on return:
- set the primitive failure code variable `primFailCode` to 0 on success, or an error code otherwise
- if failure, leave the stack untouched
- if success, pop the arguments, then push the result

The following code illustrates such concepts with `primitiveAdd`, the primitive instruction that adds two tagged small integers. `primitiveAdd` first fetches its two arguments from the top two slots of the stack.
Second, it checks if they are integer objects: if they are not, it returns indicating a failure.
Third, it sums both integer values, and in case of overflow (i.e., in case the addition cannot be represented as a tagged small integer) it returns indicating a failure.
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
The design of primitives revolves around type safety and memory safety.
Since Pharo is a dynamically-typed programming language, types are not know in advance.
Instead, types should be checked at runtime before performing dangerous operations.
In the example above, it checks the types of the arguments _and_ the non-overflow of the addition.
Primitives that load from or store into object slots, type check the receiver and index arguments, but also check that the index is within bounds.
Notice that such a design requires primitives to provide a clear separation between the _fast and slow paths_.
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
As with bytecode instructions, primitives use also table dispatch to map primitive numbers (or indexes) to primitive functions.
The code that follows shows a sketch of the primitive table.

```
	#(
		(1 primitiveAdd)
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

To execute a primitive, we fetch the primitive function from the table, then we dispatch the primitive using `perform:`.
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

When a message is sent, the method lookup searches the entire receiver's class hierarchy for a method with the same selector as the message.
If the algorithm reaches the top of the hierarchy (`nil`), then the interpreter sends the `doesNotUnderstand:` message to the receiver.
The interpreter massages the stack and the interpreter state a bit, and looks up the `doesNotUnderstand:` selector instead of the original selector.
This creates the illusion that the VM replaced the original message by a `doesNotUnderstand:` message.

Massaging the stack implies preparing it to send the `doesNotUnderstand:` message.
The thing is, the stack contains the receiver and arguments of the message but `doesNotUnderstand:` expects the reification of the original message as single argument. The original message reification is set up by:
- allocating an array to hold all the original arguments
- popping the arguments from the stack and store them in the new array
- allocating a message object and storing in it the original selector, argument array and lookup class

![Reifying the message for does not understand. %width=80&anchor=dnu_message](figures/interpreter_dnu.pdf)

Then, the `messageSelector` interpreter variable is overwritten with the `doesNotUnderstand:` selector found in the special objects array, and the lookup is restarted.

```caption=The lookup method revisited with `doesNotUnderstand:` support.
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

The VM implements also the `cannotInterpret:` hook. It is activated when the VM finds `nil` in the place of a class' method dictionary.
Similar to `doesNotUnderstand:`, `cannotInterpret:` requires massaging the stack and setting up the message reification.
However, the lookup cannot be restarted from the original class, as this will find a `nil` method dictionary again and _loop_.
Instead, `cannotInterpret:` restarts the lookup from the superclass of the class that had a `nil` method dictionary, stopping the recursion.

```caption=The lookup method.
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

In this chapter we studied how the Pharo interpreter is written.
Both bytecode and primitive instructions are implemented through method dispatch.
The key instruction in Pharo is the message send.
When a message send instruction is executed, the interpreter first looks up the method to execute in the receiver's hierarchy.
Then, if it is a primitive method, it executes the associated primitive instruction.
If the primitive instruction fails or the method is a normal method, the method is _activated_: a frame is created and the method's bytecode gets executed.