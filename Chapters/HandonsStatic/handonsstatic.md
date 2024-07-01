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

Let us check that our additions do not break the VM build.
For now we only compile the VM interpreter without the JIT compiler. 

Go to the iceberg folder in the pharo-local folder. 
```
cmake -S iceberg/pharo-vm -B build -DFLAVOUR=StackVM -DPHARO_DEPENDENCIES_PREFER_DOWNLOAD_BINARIES=TRUE
cmake --build build --target=install
```

We can now launch the resulting VM 
```

./build/build/dist/Pharo.app/Contents/MacOS/Pharo ../YourImage.image --interactive
```

@guille I got 
```
./build/build/dist/Pharo.app/Contents/MacOS/Pharo ../P11-VMSTATIC.image --interactive  [9:20:25]
Cannot locate any of #('libSDL2.dylib' 'libSDL2-2.0.0.dylib'). Please check if it installed on your system
```


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

Notice that here this is the default method passing message semantics. 








































### Better sendStaticLiteralMethodBytecode

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
