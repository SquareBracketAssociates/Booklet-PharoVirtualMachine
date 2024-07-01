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
sendLiteralSelector0ArgsBytecode
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
commonSendOrdinary
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


















