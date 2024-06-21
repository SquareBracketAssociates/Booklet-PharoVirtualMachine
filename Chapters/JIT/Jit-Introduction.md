# Introducing the Cogit JIT compiler

JIT (Just-in-Time) compilers are an optimization technique often used for interpreted languages and virtual machines.
JIT compilers detect frequently used code and compile it to more efficient code, thus reaching a good balance between execution time and compilation time. In other words, the idea is that the application run-time spends too much time compiling but executing application code instead. 
In this framework, non-frequent code falls back in slower execution engines such as interpreters.
For example, the Pharo and the Java VM run on a bytecode interpreter and eventually compile machine code for methods that are frequently called.

The current Pharo JIT compiler, namely Cogit, implements a template-based of bytecode to object code compiler.
When a method is compiled, each bytecode is mapped to a template.
In its most basic mode, all templates are concatenated to form a single machine code method.
This generated machine code method removes lots of interpretation overhead such as bytecode fetching and decoding.

This chapter starts by refreshing the reader's memory with how source code gets translated to bytecode.
Then it shows a general overview how bytecode are translated to machine code by the Cogit bytecode-to-machine-code JIT compiler.
This chapter discusses the Cogit template-based architecture.
Then it discusses the Cog register-transfer-language intermediate representation (IR), or CogRTL, used by the compiler to represent machine code before the actual machine code generation.

Finally, this chapter introduces two different compilers defined with the Cog architecture.
First, the stack-based cogit is the simplest compiler in the Cogit family.
The stack-based cogit translates each bytecode on isolation of others, and data is exchanged through the actual memory stack at run-time.
Second, the stack-to-register-mapping compiler uses a simulated stack schema to expand the compilation context of bytecode translation.
At compile-time bytecodes read and write from a simulated stack, generating code to access the real stack only if necessary.
This schema avoids useless writes and reads from memory.

## The Cogit Architecture

The Pharo Cog VM is a bytecode-based execution engine.
In other words, Pharo code is first compiled into compact and architecture-independent code called bytecode.
The bytecode is then interpreted by a bytecode interpreter, and when required compiled to machine code on-the-fly.

### The Bytecode Stack Machine

To execute Pharo source code, each method is first compiled by a bytecode compiler.
The bytecode compiler generates a method made of machine-independent bytecode and a literal frame containing all literal objects used in that method.
The bytecodes are virtual machine code instructions.
Most instructions fit in a single byte (hence their name).
The main reason of their compactness is that they all have an implicit argument: a stack of operands.
The literals are the objects that represent fixed values used in the method.

Consider for example the following method.
In this example, the method has two main operations, the message send `+` and an explicit return `^`.
This method also contains 2 literal objects: 1 and 17.
Although in this example both literals are numbers numbers, literals can be `'strings'`, literal arrays such as `#(a b c 1)` and characters `$A`. 

```smalltalk
MyClass >> foo
  ^ 1 + 17
```

To execute the method above, we need first to generate stack-based bytecode for it.
A bytecode method should first send the message `+` to the number `1` with argument `17`, and then return the result to the caller.
The send and the return operations exchange values through the operand stack.
The send operation takes the message receiver and the message arguments from the stack, executes the operation, and pushes the result to the stack.
Subsequent operations take this result from the stack if needed.
The following abstract bytecode illustrates this behaviour.

```
push 1
push 17
send +
returnTop a
```

Notice that since a send operation requires that the receiver and arguments are present in the stack, previous operations must take care of pushing such values before. In our case, this is done with explicit push operations.
In the example above, the first bytecode pushes a 1 into the stack and the second bytecode pushes a 2.
Then, the third bytecode has to send a message `+`, so it pops two elements from the stack: one to use as the receiver, and one to use as an argument.
When the execution of the message send finishes, the result is pushed to the stack.
Finally, the last bytecode takes the top of the stack (the result of the addition), and returns it to the caller.

A very simple stack-based bytecode representation of a method can be obtained with a recursive depth-first post-order traversal of the method's abstract syntax tree. Such source-code to bytecode compilation is done in Pharo with Opal.
Creating and inspecting a method like the one above shows the following in an inspector.
The first number is the bytecode index, the second is the instruction byte(s), and then there is a mnemonic of the instruction and arguments.

```
25 <76> pushConstant: 1
26 <20> pushConstant: 17
27 <B0> send: +
28 <7C> returnTop
```

### Understanding the bytecode interpreter

The `StackInterpreter` class implements Pharo's VM bytecode interpreter.
When a bytecode method is executed by the interpreter, the execution engine iterates over all bytecodes of a method and executes a VM routine for each of them. The interpreter contains a table mapping each bytecode to the routine implementing its behavior.
For example, the bytecode 0x76 is mapped to the routine `pushConstantOneBytecode`.
This routine pushes a 1 to the stack calling the `internalPush:` method.
Since pushing the value 1 is a very common operation, a special bytecode is used for it to avoid storing the 1 in the literal frame.

```smalltalk
StackInterpreter >> pushConstantOneBytecode

	self fetchNextBytecode.
	self internalPush: ConstOne.
```

For pushing less common values to the stack, bytecode 0x20 is mapped to the routine `pushLiteralConstantBytecode`.
This routine pushes a literal value stored in the method's literal frame to the stack also using `internalPush:`.
The value pushed is taken from the literal frame of the method, and the index is calculated from manipulating the `currentBytecode` variable.
Bytecode 33 pushes the first literal in the frame (33 bitAnd: 16r1F => 1), bytecode 34 pushes the second literal, and so on...

```smalltalk
StackInterpreter >> pushLiteralConstantBytecode
	<expandCases>
	self
		cCode: "this bytecode will be expanded so that refs to currentBytecode below will be constant"
			[self fetchNextBytecode.
			 self pushLiteralConstant: (currentBytecode bitAnd: 16r1F)]
		inSmalltalk: "Interpreter version has fetchNextBytecode out of order"
			[self pushLiteralConstant: (currentBytecode bitAnd: 16r1F).
			 self fetchNextBytecode]
```

Finally, bytecodes such as the message send `+` are implemented as follows.
First this bytecode gets the top 2 values from the stack.
Then it checks if boths are integers, and if the result is an integer, in which case it pushes the value and finishes.
If they are not integers, it tries to add them as floats.
If that fails, it will perform a (slow) message send using the `normalSend` method.

```smalltalk
StackInterpreter >> bytecodePrimAdd
	| rcvr arg result |
	rcvr := self internalStackValue: 1.
	arg := self internalStackValue: 0.
	(objectMemory areIntegers: rcvr and: arg)
		ifTrue: [result := (objectMemory integerValueOf: rcvr) + (objectMemory integerValueOf: arg).
				(objectMemory isIntegerValue: result) ifTrue:
					[self internalPop: 2 thenPush: (objectMemory integerObjectOf: result).
					^ self fetchNextBytecode "success"]]
		ifFalse: [self initPrimCall.
				self externalizeIPandSP.
				self primitiveFloatAdd: rcvr toArg: arg.
				self internalizeIPandSP.
				self successful ifTrue: [^ self fetchNextBytecode "success"]].

	messageSelector := self specialSelector: 0.
	argumentCount := 1.
	self normalSend
```

### (Prelude) Understanding the existing Cogit JIT compiler

When a bytecode method is executed a couple of times, the Pharo virtual machine decides to compile it to machine code.
Compiling the method to machine code avoids performance overhead due to instruction fetching, and allows one to perform several optimizations.
The compilation of a machine code method goes pretty similar to the interpretation of a method.
The JIT compiler iterates the bytecode method and for each of the bytecodes it executes a code generation routine.
This means that we will (almost) have a counterpart for each of the VM methods implementing bytecode interpretation.

For example, the machine code generator implemented for `StackInterpreter>>pushLiteralConstantBytecode` is `Cogit>>genPushLiteralConstantBytecode`.

```smalltalk
Cogit >> genPushLiteralConstantBytecode
	^self genPushLiteralIndex: (byte0 bitAnd: 31)

StackToRegisterMappingCogit >> genPushLiteralIndex: literalIndex "<SmallInteger>"
	"Override to avoid the BytecodeSetHasDirectedSuperSend check, which is unnecessary
	 here given the simulation stack."
	<inline: false>
	| literal |
	literal := self getLiteral: literalIndex.
	^self genPushLiteral: literal
```

The JIT'ted version of the addition bytecode (`genSpecialSelectorArithmetic`) is slightly more complicated, but it pretty much matches what it is done in the bytecode.

### Overview of Druid

In Druid, a meta-interpreter analyzes the bytecode interpreter code and generates an intermediate representation from it.
A compiler interface then generates machine code from the intermediate representation.
The output of the intermediate representation should have in general terms the same behaviour as the existing Cogit JIT compiler.

To verify the correctness of the compiler we use:
 - a machine code simulator (Unicorn)
 - a disassembler (llvm)

(please see the links to these projects in the references)

## The (meta-)interpreter

The setup is the following: we have one Pharo AST interpreter that we call the meta-interpreter that executes the code code of the 
`StackInterpreter` and generates the corresponding intermediate representation of `StackInterpreter` methods.
Check `Fun with interpreters` from the references to see more details on what an ASTs and abstract interpreters are.
In the code below, the meta-interpreter is called `DRASTInterpreter` (for the DruidAST interpreter) and it will analyse the methods 
of the stack interpreter returned byt the expression `Druid new newBytecodeInterpreter`.
For this task, the meta-interpreter uses an IR builder that is responsible for encapsulating the logic of IR building.

```smalltalk
builder := DRIRBuilder new.
builder isa: #X64.

astInterpreter := DRASTInterpreter new.
astInterpreter vmInterpreter: Druid new newBytecodeInterpreter.
astInterpreter irBuilder: builder.
```

This AST interpreter then receives as input a list of bytecodes to analyze, it maps each bytecode to the routine to execute, 
obtains the AST and interprets each of the instructions of the AST using a visitor pattern.

```smalltalk
astInterpreter interpretBytecode: #[76].
```

### Visiting the AST

The `DRASTInterpreter` class implements a `visiting` protocol where the `visit` methods are grouped.
Most of the visit methods are simple, like the following ones:

```smalltalk
DRASTInterpreter >> visitSelfNode: aRBSelfNode 

	^ self receiver

DRASTInterpreter >> visitTemporaryNode: aRBTemporaryNode 
	
	^ currentContext temporaryNamed: aRBTemporaryNode name
```

### Context reification

To properly analyze the scope of temporary variables, the AST interpret reifies the contexts/stack frames.
On each method or block activation, a new context is pushed to the stack.
The temporary variables are then read and written from the current context.
When a method or block returns, the current context is popped from the stack, to return to the caller context.

### Interpreting Message sends

The most important operation in a Pharo program are message sends.
Message sends are used not only for normal method invocations, but also for common operators such as additions (`+`) and multiplications (`*`) and control flow such as conditionals (`ifTrue:`) and loops (`whileTrue:`).
However, some of these special cases require special treatments when generating code.
For example, a multiplication should directly generate a normal multiplication and not require interpreting how that multiplication is implemented.

To manage such special cases, we keep a table in the interpreter that maps (special selector -> special interpretation).
When we find a message send in the AST, we lookup the selector in the table.
If we find a special case in the entry, we invoke that special entry in the interpreter.
Otherwise, we lookup the method and activate the new method, which will recursively continue the interpretation

```smalltalk
DRASTInterpreter >> visitMessageNode: aRBMessageNode 
	
	| arguments astToInterpret receiver |
	
	"First interpret the arguments to generate instructions for them.
	If this is a special selector, treat it specially with those arguments.
	Otherwise, lookup and interpret the called method propagating the arguments"
	receiver := aRBMessageNode receiver acceptVisitor: self.
	arguments := aRBMessageNode arguments collect: [ :e | e acceptVisitor: self ].

	specialSelectorTable
		at: aRBMessageNode selector
		ifPresent: [ :selfSelectorToInterpret |
			^ self perform: selfSelectorToInterpret with: aRBMessageNode with: receiver with: arguments ].

	astToInterpret := self
		lookupSelector: aRBMessageNode selector
		receiver: receiver
		isSuper: aRBMessageNode receiver isSuper.
	^ self interpretAST: astToInterpret withReceiver: receiver withArguments: arguments.
```

For example, the following code snippet shows how the `internalPush:` is mapped as just a normal push instruction in the intermediate representation.

```smalltalk
DRASTInterpreter >> initialize

	super initialize.
	irBuilder := DRIRBuilder new.
	
	specialSelectorTable := Dictionary new.
    ...
	specialSelectorTable at: #internalPush: put: #interpretInternalPushOn:receiver:arguments:.
    
DRASTInterpreter >> interpretInternalPushOn: aRBMessageNode receiver: aStackInterpreterSimulatorLSB arguments: aCollection 

    ^ irBuilder push: aCollection first
```

### The intermediate representation

The intermediate representation is generated by a builder object, instance of `DRIRBuilder`.
The AST interpreter collaborates with it to create instructions, new basic blocks and so on.
The intermediate representation that is created is somewhat inspired on a low-level intermediate representation as described in [Linear Scan Register Allocation for the Java HotSpot™ Client Compiler].

### Some meta-interpretation of special cases

Not all special cases generate instructions or interact with the DRIRBuilder.
Some of them actually modify the state of the AST interpreter.
For example, a special case is the `fetchNextBytecode` instruction, that makes the interpreter move to the next byte in the list of bytecodes.
To simulate the same behaviour in our interpreter, its special case is implemented as follows:

```smalltalk
DRASTInterpreter >> interpretFetchNextBytecodeOn: aMessageSendNode receiver: aReceiver arguments: arguments

	self fetchNextInstruction
    
DRASTInterpreter >> fetchNextInstruction

	currentBytecode := instructionStream next
```

### The compiler interface

Once the interpreter finishes its job, the irBuilder will contain the intermediate representation instructions.
We can then generate machine code from them using the `DRIntermediateRepresentationToMachineCodeTranslator` class.

```smalltalk
	astInterpreter irBuilder assignPhysicalRegisters.

	mcTranslator := DRIntermediateRepresentationToMachineCodeTranslator
		translate: builder instructions
		withCompiler: cogit.

	address := cogit methodZone freeStart.
	endAddress := mcTranslator generate.
```

`DRIntermediateRepresentationToMachineCodeTranslator` iterates all the instructions and uses the double-dispatch pattern to 
generate code for each of them.
It uses a backend to generate the actual machine code.

```smalltalk
aCollection do: [ :anIRInstruction | 
	anIRInstruction accept: self ].

DRIntermediateRepresentationToMachineCodeTranslator >> visitAdd:
DRIntermediateRepresentationToMachineCodeTranslator >> visitLoad:
DRIntermediateRepresentationToMachineCodeTranslator >> visitPush:
```

## About the Tests

The whole process has the following steps:

1. The bytecode interpreter is interpreted to generate the instructions in the IRBuilder
2. Then the physical registers will be allocated to the Druid instructions
3. The DRInstructions generates the AbstractInstructions (Cogit) that basically are one to one representations with the machine code
4. Compilation to machine code byte array

So, we have different tests to test different stages in the transformation:

- The class `DRIRBuilderTest` tests the construction of the ir.
- The class `DRASTInterpreterTest` tests the effect of interpreting ASTs
- The DRIntermediateRepresentationToMachineCodeTranslatorTest simulates that each DR instrucion is correctly translated to the corresponding machine code
- The class `DRSimulateGeneratedBytecodeTest` tests from end-to-end the generation of code of a bytecode and its execution in a machine code simulator

Other test classes test a basic register allocation algorithm (`DRRegisterAllocationTest`), the generation of machine code from an IR without passing through the AST (`DRIntermediateRepresentationToMachineCodeTranslatorTest`) and the execution of machine code from an IR without passing through the AST  (`DRSimulateGeneratedCodeTest`).

## Little exercises
- In tbe book [https://github.com/SquareBracketAssociates/PatternsOfDesign/releases](https://github.com/SquareBracketAssociates/PatternsOfDesign/releases)
- Chapter 4: Die and DieHandle double Dispatch (if you want to make sure that Double Dispatch has been understood do the Stone Paper Scissor Chapter)
- Chapter 3 A little expression interpreter
- Chapter 6 Understanding visitor 
- After reading [https://github.com/SquareBracketAssociates/Booklet-FunWithInterpreters](https://github.com/SquareBracketAssociates/Booklet-FunWithInterpreters)


## References
 - Linear Scan Register Allocation for the Java HotSpot™ Client Compiler
   http://www.ssw.uni-linz.ac.at/Research/Papers/Wimmer04Master/
 - Practical partial evaluation for high-performance dynamic language runtimes
   https://dl.acm.org/doi/10.1145/3062341.3062381
 - Structure and Interpretation of Computer Programs
   http://web.mit.edu/alexmv/6.037/sicp.pdf
 - Fun with Interpreters
   https://github.com/SquareBracketAssociates/Booklet-FunWithInterpreters/releases/download/continuous/fun-with-interpreters-wip.pdf
 - [Trace-Based Register Allocation](https://gitlab.inria.fr/RMOD/vm-papers/-/blob/master/compilation+JIT/2016_Trace-based%20Register%20Allocation%20in%20a%20JIT%20Compiler.pdf)
 - Paper explaining the motivations behind Sista Bytecode
   https://github.com/SquareBracketAssociates/Booklet-PharoVirtualMachine/raw/master/bib/iwst2014_A%20bytecode%20set%20for%20adaptive%20optimizations.pdf

 - https://github.com/unicorn-engine/unicorn
 - https://github.com/guillep/pharo-unicorn
 - http://llvm.org/
 - https://github.com/guillep/pharo-llvmDisassembler