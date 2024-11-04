## Adding a new syntax to support static calls

To be able to generate these static calls from a source program, we need to generate the right AST node at parsing.

For this we had to alter the parseKeywordMessageWith:, parseBinaryMessageWith:, and  parseUnaryMessageWith:


``` 

parseUnaryMessageWith: aNode

	| selector classTagDelimiter staticClassToCall |
	selector := self selectorNodeClass
		            value: currentToken value
		            keywordPositions: { currentToken start }.
	self step.
	^ (selector value beginsWith: '_')
		  ifFalse: [
			  self messageNodeClass
				  receiver: aNode
				  selector: selector
				  keywordsPositions: selector keywordPositions
				  arguments: #(  ) ]
		  ifTrue: [
			  classTagDelimiter := selector value allButFirst
				                       indexOf: $_
				                       ifAbsent: [ 0 ].
	
			  classTagDelimiter > 1
				  ifTrue: [
					  staticClassToCall := selector value
						                       copyFrom: 2
						                       to:
						                       (selector value allButFirst indexOf: $_).
					  (self staticMessageNodeClass
						   receiver: aNode
						   selector: (selector value: (selector value
									     copyFrom: (selector value allButFirst indexOf: $_) + 1
									     to: selector value size))
						   keywordsPositions: selector keywordPositions
						   arguments: #(  )) staticClass: staticClassToCall ]
				  ifFalse: [
					  self staticMessageNodeClass
						  receiver: aNode
						  selector: selector
						  keywordsPositions: selector keywordPositions
						  arguments: #(  ) ] ] 
						  
```

The selector of the message is checked if it begins with "_" symbol, it is then considered to be a static call. If the selector has a second underscore, it means that the programmer is defining the class in the first part and in the second a selector to look for in that class's method dictionnary. A static message node is generated and a bytecode is associated to it as described in previous parts of this experience.

this method has some limitations. For example if the declared class is not a subsequent parent of the class of the receiver, it will show strange behaviour like:
`true _asFloat` returns a float value of the address of the true Object in memory.
Further guards should be installed in the current implementation.


##Testing the static call under stackInterpreter

To be able to test that the new bytecode introduced to the StackInterpreter, we can use the VMBytecodesTest test class that sets up a simulated environment to send messages. 

```

testStaticSendMessageWithTwoArgumentsMakeAFrame

	| selectorOop aMethod aMethodToActivate receiver receiverClass aMethodDictionary arg1 arg2 |
	selectorOop := memory integerObjectOf: 42.
	
	aMethodToActivate := methodBuilder newMethod
		                     numberOfArguments: 2;
									bytecodes: #[32 "PushLiteral 1"
				92 "Return top"];
		                     buildMethod.
		methodBuilder newMethod literals: { aMethodToActivate }.
	aMethod := methodBuilder buildMethod.
	receiver := memory integerObjectOf: 41.
	receiverClass := self setSmallIntegerClassIntoClassTable.
	self setUpMethodDictionaryIn: receiverClass.
	aMethodDictionary := memory
		                     fetchPointer: MethodDictionaryIndex
		                     ofObject: receiverClass.

	self
		installSelector: selectorOop
		method: aMethodToActivate
		inMethodDictionary: aMethodDictionary.
	stackBuilder addNewFrame
		method: aMethod;
		stack: { 
				receiver.
				}.
	stackBuilder buildStack.

	interpreter methodDictLinearSearchLimit: 3.
	interpreter setBreakSelector: nil.
	interpreter method: aMethod.
	interpreter currentBytecode: 246.
	memory byteAt:   interpreter instructionPointer + 1 put:0 .
	self interpret: [ interpreter sendStaticLiteralMethodBytecode  ].

"Need to assert that the interpreter newMethod is the called method"
	self assert: interpreter stackTop equals: 1 .
	
```

In The following test case, we first define a method having standard behaviour of pushing and returning top of the stack.
Then it is assigned as a literal in a new calling method.
The method dictionary of the receiver class is set up to have the called Method, then a simulated stack frame is built for the calling method with the receiver pushed to the stack.

Set the interpreter current bytecode to 246, and the next one as 0 to match the literal offset of the method.

The assertion is based on the expected returned value of the static message send.



## Defining genSendStaticMessage
In a production virtual machine that uses StackToRegisterMappingCogit, a static message send should directly jump to compiled methods when possible, while falling back to interpreter when necessary

To be able to integrate the new bytecode we need to introduce it in 
`StackToRegisterMappingCogit >>#initializeBytecodeTableForSistaV1`. 

```
...

(2 246 246	genSendStaticMessage)
(2 247 247	unknownBytecode)

...

```

This table links the 246 bytecode with the following method: 

```
genSendStaticMessage
| methodLiteralIndex argumentCount methodToGoTo jumpInterpret mergeJump |
" i need to read literal of the stack frame, found at byte1 index, after that load it to temp, we check the method is cogged or not to jump to it via trampoline"

	methodLiteralIndex := byte1.
	
	"Fetch the method from the stack"
	methodToGoTo := self getLiteral: methodLiteralIndex .
	argumentCount := coInterpreter argumentCountOf: methodToGoTo.  

	"Allocate registers"
	self marshallSendArguments: argumentCount.

	self ssAllocateRequiredReg: ClassReg.
	self genMoveConstant: methodToGoTo R: ClassReg.
	objectRepresentation genEnsureObjInRegNotForwarded: ClassReg scratchReg: TempReg.
	
	"If the method is compiled, we jump to it, if not we jump to the interpret."
	objectRepresentation genLoadSlot: HeaderIndex sourceReg: ClassReg destReg: TempReg.
	jumpInterpret := objectRepresentation genJumpImmediate: TempReg.

	"Jump to the method's unchecked entry-point."
	self AddCq: cmNoCheckEntryOffset R: TempReg.
	self CallR: TempReg. "Receiver and Args travel in registers"
	mergeJump := self Jump: 0.
	
	"Call the trampoline to continue execution in the interpreter"
	jumpInterpret jmpTarget: self Label.
	self PushR: ReceiverResultReg.
	self CallRT: staticSendTrampoline. "Receiver and Args travel in the stack"

	mergeJump jmpTarget: self Label.
	self annotateBytecode: self Label.

	self voidReceiverOptStatus.
	self ssPushRegister: ReceiverResultReg.
	^0
	
```

We get the literal index of the method to get from the literal stack, we then count the arguments passed to the called method.
Marshall send the arguments, that makes sure to spill values in the stack that need to be saved before the call. 

Move the method object into ClassReg then check that it is actually a pointer and not forwarded, if so update the reference to point to the new location.  This assumes the object is not pointing to an immediate but rather to a compiled method. 


Now to decide wether to jump to the interpret, We need to load the method header to check if it's compiled.
then prepare a conditional jump to interpreter if not compiled.

 1- If compiled, jump directly to the method unchecked entry-point.

 2- If not compiled, set up a call to the interpreter trampoline. by pushing the receiver to the stack.
 
The two execution paths merge and the result register is pushed to the stack.


##Define a trampoline 

 A trampoline is responsible for transitioning from the native code context to interpreted mode.
 
```
ceStaticSendInterpreted: aCompiledMethodOop

	<api>
	| primitiveIndex |
	instructionPointer := self popStack.
	newMethod := aCompiledMethodOop.

	primitiveIndex := self primitiveIndexOf: newMethod.
	primitiveFunctionPointer := self functionPointerFor: primitiveIndex inClass: nil.

	self executeNewMethod: false.
	self returnToExecutive: false	

```
This trampoline pops the instruction pointer from the stack to save it, then branches to the compiled method passed to it. 
Then sets the compiled method passed to it as the next method to execute by the interpreter. then returns depending on the frame, here we are in interpreter mode so this should only return the value.

```
generateStaticSendTrampoline

	staticSendTrampoline := self
		                        genTrampolineFor: #ceStaticSendInterpreted:
		                        called: 'ceStaticSendInterpreted'
		                        arg: ClassReg                
```

##Compiling the vm

This time we compile the VM interpreter with the JIT compiler. 

- We first have to commit the changes. 

- Under the iceberg folder we build the vm taking to account the changes. 

```
cmake -S iceberg/pharo-vm -B build -DPHARO_DEPENDENCIES_PREFER_DOWNLOAD_BINARIES=TRUE
```

Then we compile the Vm and the result will be in the build folder. 

```
cmake --build build --target=install
```

We can now launch it using:

```
./build/build/dist/Pharo.app/Contents/MacOS/Pharo ../YourImage.image --interactive
```

##Testing the new genSendStaticMessage bytecode

We define our tests in VMStackToRegisterMappingCogitTest

```
VMStackToRegisterMappingCogitTest subclass: #VMStackToRegisterMappingStaticMessageSendTest
	instanceVariableNames: ''
	classVariableNames: ''
	package: 'VMMakerTests-JitTests' 
```

adding the following method 

```
initStack

	self createBaseFrame.
	
	"Initialize Stack to the correct pointers in the selected page"
	machineSimulator smalltalkStackPointerRegisterValue: interpreter stackPointer.
	machineSimulator framePointerRegisterValue: interpreter framePointer.
	machineSimulator baseRegisterValue: cogit varBaseAddress.

	cogit setCStackPointer: interpreter rumpCStackAddress.
	cogit setCFramePointer: interpreter rumpCStackAddress.

```
To initialize the stack for simulation.

Then redefine setUpTrampolines  

```
setUpTrampolines

	super setUpTrampolines.
	cogit generateStaticSendTrampoline.

	cogit methodAbortTrampolines at: 0 put: cogit ceMethodAbortTrampoline.
	cogit methodAbortTrampolines at: 1 put: cogit ceMethodAbortTrampoline.
	cogit methodAbortTrampolines at: 2 put: cogit ceMethodAbortTrampoline.
	cogit methodAbortTrampolines at: 3 put: cogit ceMethodAbortTrampoline.
	
	cogit picMissTrampolines at: 0 put: cogit ceCPICMissTrampoline.
	cogit picMissTrampolines at: 1 put: cogit ceCPICMissTrampoline.
	cogit picMissTrampolines at: 2 put: cogit ceCPICMissTrampoline.
	cogit picMissTrampolines at: 3 put: cogit ceCPICMissTrampoline.

	cogit picAbortTrampolines at: 0 put: cogit cePICAbortTrampoline.
	cogit picAbortTrampolines at: 1 put: cogit cePICAbortTrampoline.
	cogit picAbortTrampolines at: 2 put: cogit cePICAbortTrampoline.
	cogit picAbortTrampolines at: 3 put: cogit cePICAbortTrampoline.
	
	
		
	cogit ceStoreCheckTrampoline: (self compileTrampoline: [ cogit RetN: 0 ] named:#ceStoreCheckTrampoline).
	cogit objectRepresentation setAllStoreTrampolinesWith: (self compileTrampoline: [ cogit RetN: 0 ] named: #ceStoreTrampoline).
```

this generates the trampoline for static send and sets method abort and PIC miss/abort trampolines.

```

testStaticSendOfCoggedMethodCallMethodDirectly

	| selector calledCompiledMethod calledCogMethod staticCallerCompiledMethod staticCallerCogMethod |

	self setUpTrampolines.
	cogit computeEntryOffsets.

	calledCompiledMethod := self createMethodOopFromHostMethod: (Object >> #yourself).
	calledCogMethod := cogit cog: calledCompiledMethod selector: memory nilObject.
	
	selector := self newOldSpaceObjectWithSlots: 0.
	
	staticCallerCompiledMethod := methodBuilder newMethod
			 literalAt: 0 put: (memory integerObjectOf: 13);
			 literalAt: 1 put: calledCompiledMethod;
			 literalAt: 2 put: selector;
			 literalAt: 3 put: memory nilObject;
			 bytecodes: #[32 "PushLiteral 1" 
				246 1 "Static Send literal at 2"
				92 "Return top"];
			 buildMethod.

	staticCallerCogMethod := cogit cog: staticCallerCompiledMethod selector: memory nilObject.

	self createFramefulCallFrom: callerAddress.
	
	"Push receiver, arg, then send"
	self pushAddress: memory falseObject.

	self 
		runFrom: (staticCallerCogMethod address + cogit noCheckEntryOffset) 
		until: calledCogMethod address + cogit noCheckEntryOffset. 

	self assert: machineSimulator pc equals: calledCogMethod address + cogit noCheckEntryOffset

```

We follow the same principle as before for testing this new bytecode in the stackToRegisterMapping by creating a method pointer from an existing method `yourself`, then compile it. We put the called compiled method as a literal of a new calling method. 

The calling method is cogged too in this test case to check that message send, correctly jumps to the machine code of the called method.

another test case would be to see if the static call to a non-compiled method would branch to the correct trampoline, checking further than trampoline is not possible under this simulation.

```
testStaticSendOfNonCoggedMethodCallsTrampoline

	| selector calledCompiledMethod staticCallerCompiledMethod staticCallerCogMethod |

	self setUpTrampolines.
	cogit computeEntryOffsets.

	calledCompiledMethod := self createMethodOopFromHostMethod: (Object >> #yourself).
	
	selector := self newOldSpaceObjectWithSlots: 0.
	
	staticCallerCompiledMethod := methodBuilder newMethod
			 literalAt: 0 put: (memory integerObjectOf: 13);
			 literalAt: 1 put: calledCompiledMethod;
			 literalAt: 2 put: selector;
			 literalAt: 3 put: memory nilObject;
			 bytecodes: #[32 "PushLiteral 1" 
				246 1 "Static Send literal at 2"
				92 "Return top"];
			 buildMethod.

	staticCallerCogMethod := cogit cog: staticCallerCompiledMethod selector: memory nilObject.

	self createFramefulCallFrom: callerAddress.
	
	"Push receiver, arg, then send"
	self pushAddress: memory falseObject.
	
	self 
		runFrom: (staticCallerCogMethod address + cogit noCheckEntryOffset) 
		until: cogit staticSendTrampoline. 
	self assert: machineSimulator pc equals: cogit staticSendTrampoline.
	self assert: machineSimulator classRegisterValue equals: calledCompiledMethod 

```
The two assertions are:

	1- The program counter is expected to be at the end address of the runFrom:until:.

	2- The method object should still be in the class register from the previous static send.

## slowFactorial speed up

To showcase the effect of the implementation, we take the **slowFactorial** since it is a recursive method, and refactor the recursive call in it as well as the call site.

```
[6 slowFactorial] bench. 182,160,516 iterations in 5 seconds 2 milliseconds. 36417536.186 per second.
[6 _Integer_slowFactorial] bench. 1,747,933 iterations in 5 seconds 3 milliseconds. 349376.974 per second

[180 slowFactorial] bench.  234,222 iterations in 5 seconds 1 millisecond. 46835.033 per second
[180 _Integer_slowFactorial] bench  56,620 iterations in 5 seconds 2 milliseconds. 11319.472 per second

[ 1600 slowFactorial ] bench. 4,938 iterations in 5 seconds 3 milliseconds. 987.008 per second
[ 1600 _Integer_slowFactorial ] bench  2,847 iterations in 5 seconds 5 milliseconds. 568.831 per second.

```

##Limits and conclusion

 The static calls are much faster than normal sends. However current implementation is still lacking:
	- Type mismatch handling. 
	- Invalidation in case the called method changes.

A next step is to benchmark test methods that either call directly or indirectly methods with a single implementation in the image ( monomorphic call sites ), by prefixing them with class tags following the new syntax.