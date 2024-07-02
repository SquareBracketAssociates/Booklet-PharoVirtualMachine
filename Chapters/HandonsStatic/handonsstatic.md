## Adding Static Methods


In this chapter we will show how you can develop a prototype version of static methods in Pharo. 
A static method is a method with no lookup. It means that the call site defines the exact class
where the method is defined. The VM has just to grab the method from such a class.

For this introduction, we will
- define a new bytecode
- extend the bytecode builder
- extend the bytecode interpreter to handle this bytecode

### Bytecode table

Since we will add a bytecode we start to have a look at the bytecode table.
You will find it in the method `StackInterpreter class >> #initializeBytecodeTableForSistsV1`

This table links a bytecode and the method that defines its behavior. 

What you can see is that the bytecodes 246 and 247 are free. 

```
...
		(243		extStoreReceiverVariableBytecode)
		(244		extStoreLiteralVariableBytecode)
		(245		longStoreTemporaryVariableBytecode)

		(246 247	unknownBytecode)

		"3 byte bytecodes"
		(248		callPrimitiveBytecode)
		(249		extPushFullClosureBytecode)
...		
```

We will use the bytecode 246. 
Once we will have extended the interpreter we will come back and modify such a table. 


### About method execution

Let us study a bit the normal message send bytecodes.
For a default late bound message 
- the receiver and args are on the stack
- the method selector is stored in the method literal frame

```

		(128 143	sendLiteralSelector0ArgsBytecode)
		(144 159	sendLiteralSelector1ArgBytecode)
		(160 175	sendLiteralSelector2ArgsBytecode)

```

The table tells us that send bytecodes range from 128 to 175.
Such bytecodes are compact in the sense that they encode their number of arguments. 
In addition, they encode the place in the literal frame where the selector of the method to be looked up 
is placed. 

For example, 128 means that the selector is located in the first place of the literal frame.

```
128 bitAnd: 16rF
> 0
```

### Study 128 

The interpreter method associated to bytecode 128 is `sendLiteralSelector0ArgsBytecode`

```
StackInterpreter >> sendLiteralSelector0ArgsBytecode
	"Can use any of the first 16 literals for the selector."
	
	| rcvr |
	messageSelector := self literal: (currentBytecode bitAnd: 16rF).
	argumentCount := 0.
	rcvr := self stackValue: 0.
	lkupClassTag := objectMemory fetchClassTagOf: rcvr.
	self assert: lkupClassTag ~= objectMemory nilObject.
	self commonSendOrdinary
```

What we see is that 
- The message selector is extracted from literal frame using the the bytecode encoding. 
- Then it sets the number of argument, here to zero
- It then looks the class up. 
- And finally executes the method `commonSendOrdinary`


```
StackInterpreter >> commonSendOrdinary
	"Send a message, starting lookup with the receiver's class."
	"Assume: messageSelector and argumentCount have been set, and that 
	the receiver and arguments have been pushed onto the stack,"
	"Note: This method is inlined into the interpreter dispatch loop."
	
	<sharedCodeInCase: #extSendBytecode>
	self sendBreakpoint: messageSelector receiver: (self stackValue: argumentCount).
	self doRecordSendTrace.
	self findNewMethodOrdinary.
	self executeNewMethod: false.
	self fetchNextBytecode
```

Once the method is found, it is executed by `executeNewMethod: false`.
The argument means that the method should not be compiled by the JIT compiler. 
Then the interpreter fetches the next bytecode to be executed.

### A first version of sendStaticLiteralMethod

The bytecode 246 is a two byte bytecode. 
Let us start to define a new method `sendStaticLiteralMethodBytecode` that defines the behavior of the static send. Since we want to avoid performing a method lookup we decide that the compiled method
the send will execute should be stored in the method literal frame.


```
StackInterpreter >> sendStaticLiteralMethodBytecode
	"two bytecodes
		opecode 
		literal offset "
	| methodLiteralOffset |
	methodLiteralOffset:= self fetchByte.
	newMethod := self literal: methodLiteralOffset.
	self executeNewMethod: true. "could be compiled"
	self fetchNextBytecode
```

This is a first version because the interpreter may use the values of other global variable such as the argument count. We will refine this definition later. 

Now we declare that the bytecode 246 is defined by `sendStaticLiteralMethod`

```
...
		(243		extStoreReceiverVariableBytecode)
		(244		extStoreLiteralVariableBytecode)
		(245		longStoreTemporaryVariableBytecode)

		(246 		sendStaticLiteralMethodBytecode)
		(247		unknownBytecode)

		"3 byte bytecodes"
		(248		callPrimitiveBytecode)
		(249		extPushFullClosureBytecode)
...		
```

### Compiling the VM


Let us check that our additions do not break the VM build - so far we nearly do anything that could but this way you can practice.
Note that we only compile the VM interpreter without the JIT compiler. 

- First save your code, the build will take the current branch has input. 

- Go to the iceberg folder in the pharo-local folder and execute the following. Here we asked to grab the binaries of external projects to make the compilation faster. 

```
cmake -S iceberg/pharo-vm -B build -DFLAVOUR=StackVM -DPHARO_DEPENDENCIES_PREFER_DOWNLOAD_BINARIES=TRUE
```

Then we compile the Vm and the result will be in the build folder. 

```
cmake --build build --target=install
```

We can now launch the resulting VM to execute your image as follows: 
```

./build/build/dist/Pharo.app/Contents/MacOS/Pharo ../YourImage.image --interactive
```

Note that you will have to rebuild the VM in the following. 
Before recompiling to not forget to save your code and remember that the build is taking the current branch as input.

### Getting a compiled method

In this hands on, we focus on the virtual machine logic therefore we do not want to modify the syntax of Pharo. Still we need a way to get compiled methods with the new bytecode. 

The Pharo compiler supports a bytecode builder, using the pragma `opalBytecodeMethod` 
we can create the body of a method has the compiler would do. 

For example the following method `exampleIRBuilder` just returns 2.

```
MyXP >> exampleIRBuilder 

	<opalBytecodeMethod>
		^ IRBuilder buildIR: [:builder | 
			builder 
				pushLiteral: 2;
				returnTop ]
```

Now we can just execute the method. 

```
MyXP new exampleIRBuilder
> 2
``` 

Here is the definition of `factorial`, we call it `lateBoundFactorial` since we will define alternate versions using static message sends later.

```
Integer >> lateBoundFactorial

	<opalBytecodeMethod>

	^ IRBuilder buildIR: [ :builder |
		  builder
			  pushReceiver;
			  pushLiteral: 1;
			  send: #'<=';
			  jumpAheadTo: #gogogo if: false;

			  "Base case"
			  pushLiteral: 1;
			  returnTop;

			  "Recursive case"
			  jumpAheadTarget: #gogogo;
			  pushReceiver;
			  pushReceiver;
			  pushLiteral: 1;
			  send: #-;
			  send: #lateBoundFactorial;
			  send: #*;
			  returnTop ]
```

Notice that here this is the default method passing message semantics. 

To support static calls, we will define a new IR instruction in addition to the bytecode to be able to define static sends. 

### Fixing some Pharo logic

Before continuing we should fix the method `refersToLiteral:` because it can loop 
if if literal frame contains a compiled method and this is what we want to do for our solution.


```
CompiledCode >> refersToLiteral: aLiteral [
	"Answer true if any literal in this method is literal,
	even if embedded in array structure."

	1 to: self numLiterals - self literalsToSkip do: [ :index | "exclude selector or additional method state (penultimate slot)
		and methodClass or outerCode (last slot)"
		(self literalAt: index) == self ifFalse: [
			((self literalAt: index) refersToLiteral: aLiteral) ifTrue: [
				^ true ] ] ].

	^ false
]
```

### Extending the IRBuilder

We extend the builder with the new message `sendStatic:`. 


```
IRBuilder >> sendStatic: aMethod
	^ self add: (IRSendStatic sendStatic: aMethod )
```

We define a new instruction subclass of `IRInstruction`.
This instruction will refer to the invoked method.

```
IRInstruction << #IRSendStatic
	slots: { #calledMethod };
	tag: 'IR-Nodes';
	package: 'OpalCompiler-Core'
```

```
IRSendStatic >> calledMethod
	^ calledMethod
```

```
IRSendStatic >> calledMethod: aCompiledMethod
	calledMethod := aCompiledMethod
```

We also define a class method 

```
IRSendStatic class >> 
```

We define the corresponding methods to support the interaction with the Visitors who are responsible for
compilation.

```
IRSendStatic >> accept: visitor

	visitor visitStaticSend: self
```

```
Visitor >> visitStaticSend: anIRStaticSend

	self subclassResponsibility
```

```
IRPrinter >> visitStaticSend: send

 	stream nextPutAll: 'staticSend: '.
	send calledMethod selector printOn: stream.
```			


### Study the Translator

Before we extend the code translator to generate the adequate bytecode,
let us get inspiration from the method `visitSend:`.

We see that `visitSend:` is basically `gen send: send selector`.

```
IRTranslator >> visitSend: send

	send superOf
		ifNil: [ gen send: send selector ]
		ifNotNil: [ :behavior |  gen send: send selector toSuperOf: behavior ]
```

The `send:` method of the `IRBytecodeGenerator` uses the selector to send adequate information to the bytecode encoder. 

```
IRBytecodeGenerator >> send: selector
	| nArgs |
	nArgs := selector numArgs.
	stack pop: nArgs.
	...
	encoder genSend: (self literalIndexOf: selector) numArgs: nArgs
```

The method `send:` is basically a send to `genSend:numArgs:`.

```
EncoderForSistaV1 >> genSend: selectorLiteralIndex numArgs: nArgs

	...
	(selectorLiteralIndex < 16 and: [nArgs < 3]) ifTrue:
	 	["128-143	1000 iiii			Send Literal Selector #iiii With 0 Argument
		  144-159	1001 iiii			Send Literal Selector #iiii With 1 Arguments
		  160-175	1010 iiii			Send Literal Selector #iiii With 2 Arguments"
		 stream nextPut: 128 + (nArgs * 16) + selectorLiteralIndex.
		 ^ self].
	...

```

### Translator extension

We make sure that the IRFix visitor does not raise an error by defining the method `visitStaticSend:` doing nothing.

```
IRFix >> visitStaticSend: anIRStaticSend
```

We extend the translator by adding the following `visitStaticSend:`

```
IRTranslator >> visitStaticSend: anIRStaticSend

	gen sendStatic: anIRStaticSend calledMethod
```

We define the method `sendStatic:`as follows: 

```
IRBytecodeGenerator >> sendStatic: aMethod

	| nArgs |
	nArgs := aMethod numArgs.
	stack pop: nArgs.
	encoder genSendStatic: (self literalIndexOf: aMethod)
```


We finally emit the new bytecode: it basically emits the bytecode 246 followed by the literal frame offset in which the compiled method is stored. 
A better version should do a bit of validation. 

```
EncoderForSistaV1 >> genSendStatic: methodLiteralOffset

	stream
		nextPut: 246;
		nextPut: methodLiteralOffset
```

### Testing

Now we define a simple method using a static send. This method adds one to the receiver. 


```
Integer >> staticPlus 

	<opalBytecodeMethod>

	^ IRBuilder buildIR: [ :builder |
		  builder
			  pushReceiver;
			  pushLiteral: 1;
			  sendStatic: (SmallInteger >> #'+');
			  returnTop ]
```

```
1 staticPlus
> 2
```




### The case of recursion

Since we are compiling a recursive method (factorial), we need a way so that the literal frame of this method refers to the compiled method itself. 

For this as a temporarily solution we will introduce a placeholder that later we will patch. 
Here is a definition of factorial where the recursive call is static. 

```
Integer >> staticBoundRecursiveFactorial

	<opalBytecodeMethod>
	1halt.
	^ IRBuilder buildIR: [ :builder |
		  builder
			  pushReceiver;
			  pushLiteral: 1;
			  send: #'<=';
			  jumpAheadTo: #gogogo if: false;

			  "Base case"
			  pushLiteral: 1;
			  returnTop;

			  "Recursive case"
			  jumpAheadTarget: #gogogo;
			  pushReceiver;
			  pushReceiver;
			  pushLiteral: 1;
			  send: #-;
			  sendStatic: (StaticRecursiveMethodPlaceHolder new selector: #staticBoundRecursiveFactorial);
			  send: #*;
			  returnTop ]
```

We have to define the class `StaticRecursiveMethodPlaceHolder`.

```
Object << #StaticRecursiveMethodPlaceHolder
	slots: {#selector};
	...
```

```
StaticRecursiveMethodPlaceHolder >> numArgs 

	^ selector numArgs
```

```
StaticRecursiveMethodPlaceHolder >> selector: aString
	selector := aString
```


Now we patch the `generate:` method to substitute the placeholder by the compiled method.

```
IRMethod >> generate: trailer

	| irTranslator |
   irTranslator := IRTranslator context: compilationContext trailer: trailer.
	irTranslator
		visitNode: self;
		pragmas: pragmas.
	compiledMethod := irTranslator compiledMethod.
	compiledMethod literals doWithIndex: [ :e :index |
		(e isKindOf: StaticRecursiveMethodPlaceHolder)
			ifTrue: [ compiledMethod literalAt: index put: compiledMethod ] ].
	self sourceNode
		ifNotNil: [
			compiledMethod classBinding: self sourceNode methodClass binding.
			compiledMethod selector: self sourceNode selector ]
		ifNil: [
			compiledMethod classBinding: UndefinedObject binding.
			compiledMethod selector: #UndefinedMethod ].
	^ compiledMethod
```	



























### Better sendStaticLiteralMethodBytecode

The first definition of `sendStaticLiteralMethodBytecode` was brittle. 
Indeed the interpreter has some invariants and use about global variables that we did not
reset correctly. 

This is the case for `argumentCount`. It is used by primitives to
check how to access the stack and know how many elements to pop, and generally to check that the stack gets balanced after execution.

The second case is `primitiveIndex` and `primitiveFunctionPointer`.
`primitieIndex` should be loaded for each interpreted method.
The function `executeNewMethod:` assumes that this index is set during lookup.
Thus, if we don't set it, the value will be the one of the last method/primitive called leading to strange bugs.

Here is the new version of the static send bycode logic. 

```
StackInterpreter >> sendStaticLiteralMethodBytecode

	"2 Byte Bytecode
		1st Byte: opcode
		2nd Byte: literal offset of the method"

	| methodLiteralOffset primitiveIndex |
	methodLiteralOffset := self fetchByte.
	newMethod := self literal: methodLiteralOffset.

	"argumentCount is used by primitives to
	   - check how to access the stack and
	   - know how many elements to pop,
	and generally to check that the stack gets balanced after execution"
	argumentCount := self argumentCountOf: newMethod.

	"primitiveFunctionPointer needs to be loaded for each method interpreted.
	executeNewMethod: assumes that this is set during lookup
	Thus, if we don't set it, the value will be the one of the last method/primitive called"
	primitiveIndex := self primitiveIndexOf: newMethod.
	primitiveFunctionPointer := self functionPointerFor: primitiveIndex inClass: nil.

	self executeNewMethod: true.
	self fetchNextBytecode
```

https://github.com/evref-inria/pharo-vm/pull/2/files

### Bench

Here are the three different versions of factorial that we can now benchmark.

```
Integer >> lateBoundRecursiveFactorial

	<opalBytecodeMethod>
	^ IRBuilder buildIR: [ :builder |
		  builder
			  pushReceiver;
			  pushLiteral: 1;
			  send: #'<=';
			  jumpAheadTo: #gogogo if: false;

			  "Base case"
			  pushLiteral: 1;
			  returnTop;

			  "Recursive case"
			  jumpAheadTarget: #gogogo;
			  pushReceiver;
			  pushReceiver;
			  pushLiteral: 1;
			  send: #-;
			  send: #lateBoundRecursiveFactorial;
			  send: #*;
			  returnTop ]
```

```
Integer >> staticBoundRecursiveFactorial

	<opalBytecodeMethod>
	1halt.
	^ IRBuilder buildIR: [ :builder |
		  builder
			  pushReceiver;
			  pushLiteral: 1;
			  send: #'<=';
			  jumpAheadTo: #gogogo if: false;

			  "Base case"
			  pushLiteral: 1;
			  returnTop;

			  "Recursive case"
			  jumpAheadTarget: #gogogo;
			  pushReceiver;
			  pushReceiver;
			  pushLiteral: 1;
			  send: #-;
			  sendStatic: (StaticRecursiveMethodPlaceHolder new selector: #staticBoundRecursiveFactorial);
			  send: #*;
			  returnTop ]
```

```
Integer >> staticBoundRecursiveFactorialHardcore

	<opalBytecodeMethod>
	^ IRBuilder buildIR: [ :builder |
		  builder
			  pushReceiver;
			  pushLiteral: 1;
			  sendStatic: (SmallInteger >> #'<=');
			  jumpAheadTo: #gogogo if: false;

			  "Base case"
			  pushLiteral: 1;
			  returnTop;

			  "Recursive case"
			  jumpAheadTarget: #gogogo;
			  pushReceiver;
			  pushReceiver;
			  pushLiteral: 1;
			  sendStatic: (SmallInteger >> #'-');
			  sendStatic: (StaticRecursiveMethodPlaceHolder new selector: #staticBoundRecursiveFactorial);
			  sendStatic: (SmallInteger >> #'*');
			  returnTop ]
```

Need more discussions

```
[17 lateBoundRecursiveFactorial.] bench. "'2774597.961 per second'"
[17 staticBoundRecursiveFactorial.] bench.  "'3693598.280 per second'"
[17 staticBoundRecursiveFactorialHardcore.] bench. "'2170939.636 per second'"
```
Note that the staticBoundRecursiveFactorialHardcore is slower because the primitives *, -, +  are extremely optimized by the VM and do not result in message sends.


### Limits and conclusion

There is clearly some more effort to obtain a full working solution. For example, managing the code changes and recompilation of the methods.

This tutorial misses
- syntax support
- JIT support
- invalidation if the called method changes
- indirect recursion support