## Pharo Specific Interpreter Optimizations

The Pharo interpreter has two main sources of execution overhead arising from its execution model.
A quick look at the standard library in Pharo 12 shows that bytecodes in a method are organized in roughly 30% of message sends and 70% of other operations such as loads, stores and stack manipulations.
On the one hand, message sends are expensive operations: a lookup that traverses the class hierarchy, performs a hash table lookup in each method dictionary, and once a method is found there is an activation in the call stack.
On the other hand, non message sends are very inexpensive operations: they typically are in the order of one or two machine instructions.
However, while this is a positive feature, it puts a lot of stress on the interpreter instruction dispatch: if the code to fetch and execute the next instruction is very expensive, then most of the time is spent dispatching instructions and not performing real work.

Optimizing these aspects require radically different approaches.
In this chapter we will explore several optimizations appearing in the interpreter itself.
Many of these optimizations target message sends: the global lookup cache, static type predictions and frameless methods.
Our last optimization, instruction lookaheads, is designed to improve instruction dispatch.

Moreover, the Pharo VM includes another major optimization to improve instruction dispatch in the general case: threading.
However, this optimization is performed automatically by the VM framework, Slang, and thus treated separately in another chapter.

### Where does the interpreter time go?

Let us compute the ratio of message send vs non message send operations.
We can quickly estimate such a ratio by taking all compiled method instances, and counting the send bytecodes in them.
We use for this the symbolic bytecode facility that *disassembles* the bytecode and gives us a textual representation.

```smalltalk
CompiledMethod allInstances inject: { 0 . 0 } into: [ :accum :each |
	| symbolicBytecodes messages counts |
	symbolicBytecodes := each symbolicBytecodes.
	messages := symbolicBytecodes count: [ :e | e description beginsWith: 'send' ].
	counts := { messages . symbolicBytecodes size - messages }.
	accum + counts ]
````

In a fresh Pharo 12, the script above returned me: `#(672418 1579103)`.
Which means that a rough 30% of bytecode instructions are message sends.

Of course such static analysis is very simplistic as it does not really give us a precise view on how much these instructions *execute*.
A more precise approach would consider that instructions in a loop will have more weight in the execution.
Also, this estimation considers all methods, while only a subset of these are realistically executed in a real program.
Anyways, although this estimation is not as precise as we would like it to be, it is nevertheless very valuable: crafting this script and computing it took us just some seconds.

Why did we differentiate message sends from other instructions?

#### Analyzing Message Sends

Message sends are of a more complex nature than the rest of the instructions, by orders of magnitude.
Let us consider a simple message send:

```smalltalk
a + b
```

When executing this message send, the interpreter will execute a lookup and a then perform a method activation.

```smalltalk
method := interpreter lookup: #+ receiver:a.
interpreter activateMethod: method receiver:a arguments: #(b).
```

The lookup operation traverses the class hierarchy of the receiver and looks for a method with the given selector in each method dictionary using a hash lookup.
The method lookup takes a long time, even in the best cases, when compared with a normal function call.
This means somehow that the execution time of a message send operation is *dominated* by the method lookup.

In the rest of this chapter we will study three interpreter optimizations to improve this situation, all based on a single premise: "The best way to optimize message sends is to not do them at all".
We can interpret this premise in two ways: 

- avoiding the expensive lookup if possible
- avoiding method activations in possible

#### Analyzing Non Message Sends

Let us now consider a simple temporary variable assignment operation in Pharo:

```smalltalk
| a |
a := b.
```

When executing such an instruction, the interpreter will just perform a store in the corresponding temporary variable slot of the current frame, which will pressumably be very fast.
For example, if we consider that variable `a` is the first temporary slot of the frame, the interpreter code will look like the following.

```smalltalk
interpreter itemporary: 1 in: theFP put: b
```

Althogh such an instruction is fast in isolation, let us now further imagine a program containing lots of them.

```smalltalk
| a1 a2 a3 ... an |
a1 := b1.
a2 := b2.
a3 := b3.
...
an := bn.
```

Of course, such a program is not be realistic, but it illustrates a further problem with interpreters: the cost of instruction dispatch.
Remember that a simple implementation of an interpreter is made of `while` and `switch` statements:

```smalltalk
Interpreter >> interpret
	...
	[ true ] whileTrue: [
		self fetchNextBytecode.
		self dispatchOn: currentBytecode in: BytecodeTable
	].
	...

Interpreter >> dispatchOn: anInteger in: selectorArray

	self perform: (selectorArray at: (anInteger + 1)).
```

This means that after each instruction is executed, a new instruction is fetched, the new instruction code is fetched and then executed.
In the example shown above, such dispatching works by means of a reflective `perform:` operation.
Overall, this means that for every instruction, and in particular for each one of our cheap assignment instructions we:

- jump back to the beginning of the loop
- fetch from memory the next instruction
- lookup the instruction code in the table
- dispatch the execution to the instruction code

If we again perform some naïf estimations, if we consider that each of these steps takes roughly the same time of our bytecode instruction, this means we have 4x overhead just because of the instruction dispatch.
Later on this chapter we will study instruction lookahead, an interpreter optimization that improves this situation by avoiding unnecessary instruction dispatches.

### Optimization Philosophy and Performance Mantras

In the next couple of sections, we are going to study several optimizations done at the level of the interpreter to solve the issues mentioned before.
Although the techniches differ, they share some common design, or better saying, a meta design.
In other words, all of them share the same the general philosophy: don't do it.

This kind philosophy is well (and funnily) summarized in Brendan Gregg's book System Performance, as the *how to improve performance checklist*.
From most to least effective:

- Don't do it
- Do it, but don't do it again
- Do it less
- Do it later
- Do it when they are not looking
- Do it concurrently
- Do it more cheaply

The Pharo Virtual Machine, its interpreter, its compiler, its runtime, follow these mantras dearly.
Lookup caches avoid double work.
Special cases avoid expensive general work for simple computations.
Baseline JIT compilation removes interpreter overhead.

### Global Lookup Cache

### Static Type Predictions

As we stated before, the faster way to perform a message send is not doing it.
However, avoiding message sends all the way is not an easy task for an interpreter.
Because of the dynamically-typed nature of Pharo programs, we do not know beforehand the type of a message receiver, and thus neither the method to activate.
Thus, the interpreter must be ready for the worst case: types and methods are unknown and a lookup must be done.
Nevertheless, although Pharo programs are dynamic in nature, this does not mean that they are not *predictable* up to a certain extent.

Static type predictions is a message send optimization that exploits such observation.
The goal of the optimization is to *inline* common and predictable message sends in a fast path guarded by a type check.
If the type check guard fails, the general case is implemented in a slow execution path.
The general expectation is that common message sends will be sped up, while rare ones will be slightly penalized with a type check.
Static type predictions were first described for the Self programming language by Holzle in his PhD thesis {!citation|ref=Holz94a!}.
Nowadays, Pharo uses this technique to optimize for common low-level integer operations such as arithmetics, comparisons and bitwise manipulations.

To illustrate static type predictions let us consider the addition operator in Pharo: `a + b`.
The addition operator is a standard binary selector, that can be redefined by developers in their own classes.
In other words, the addition operator can be overloaded, and determining if it is a user-defined addition or a system-defined addition requires a method lookup.
Finally, let us consider that:

- although possible, such overloading is rare
- even when overloading is present, operating with numbers will be much more common
- and operating with integers is potentially more common than with floats (think loops and indexed variable accesses)
- in addition, users will have higher performance expectations on number arithmetics

We optimize integer additions with static type predictions by defining a new *addition* bytecode instruction separate from the general case.
This new instruction will have two paths differentiated by an integer type check on both operands.
The fast path will inline the addition implementation if the check is a success, and perform a slow message send in case it fails.
Additionally, in case of overflow, the Pharo semantics is to activate a user defined method, to *e.g.,* raise an exception.
Thus, the slow path is taken also in case an overflow is detected.
An excerpt of the code of this bytecode is shown below.

```caption=The lookup method revisited with `doesNotUnderstand:` support.
Interpreter >> bytecodeAdd
	| rcvr arg result |
	rcvr := self stackValue: 1.
	arg := self stackValue: 0.

	"Type check both operands"
	(objectMemory areIntegers: rcvr and: arg) ifTrue: [
		result := (objectMemory integerValueOf: rcvr) + (objectMemory integerValueOf: arg).
		"Type check the result.
		If overflow, roll back to the slow path"
		(objectMemory isIntegerValue: result) ifTrue: [
			self pop: 2 thenPush: (objectMemory integerObjectOf: result).
			^ self fetchNextBytecode ]].

	"Slow path: send"
	messageSelector := self specialSelector: 0.
	argumentCount := 1.
	self normalSend
```

### Frameless Methods

### Super Instructions through Instruction Lookaheads

We saw in previous sections that high instruction dispatch overhead appears when the instruction set (the bytecode) contains instructions that are *too granular*.
That is, when the cost of dispatching in the same order of magnitude than the instruction itself.
We said in the introduction that such an overhead can be lowered in the general case through techniques such as *threading* that will be explored in later chapters.
However, part of this overhead can be solved from the design of the instruction set and the instruction implementation.

For example, let us consider the following bytecode sequence containing a comparison and a conditional jump.
Let us also consider that such a bytecode sequence is common in existing code.

```
push a
push b
send <
jumpIfTrue label
...

label:
...
```

In this scenario, one way to reduce instruction dispatch is to statically replace common sequences of instructions by single instructions.
We saw in a previous chapter that one way to solve this issue is to use super instructions.
The Pharo bytecode set contains many super instructions for common sequences of stack operations.
The advantage of such super instructions is that instruction dispatch is cut by design, and instruction size may be reduced too.
However, introducing super instructions in such a way is a costly design decision: the instruction set needs to be modified, together with the compiler and debugger, and all other tooling working on it.
Moreover, each combination of instructions takes extra space in the instruction set.

This section covers a different way to implement super instructions: the first instruction of a sequence looks ahead for the following instruction and executes the fast path (the super instruction code) if the next instruction makes part of the optimized sequence.
This way, the instruction set does not need to change, only the interpreter or the language implementation.
The trick to optimize the case above, is to mix it with static type predictions: we combine both instructions if the first send can be statically predicted for integers.
This way, in the common case where we are comparing integers, we avoid the message send and the dispatch to the jump instruction altogether.

In this example, we first define a new *comparison* bytecode instruction separate from the general send case.
This new instruction will have two paths differentiated by an integer type check on both operands.
The fast path will inline the comparison implementation if the check is a success, and perform a slow message send in case it fails.
Additionally, in the this path, a second fast path appear: if the next instruction in the instruction stream is the expected `jump` instruction a fast inlines the implementation of the jump instruction.
In case the instruction is not followed by an expected jump instruction, the result of the comparison is pushed to the stack and the message send is avoided completely.

```caption=The lookup method revisited with `doesNotUnderstand:` support.
Interpreter >> bytecodePrimLessThan
	| rcvr arg aBool |
	rcvr := self stackValue: 1.
	arg := self stackValue: 0.
	(objectMemory areIntegers: rcvr and: arg) ifTrue: [ | next |
		next := self fetchByte. "assume next bytecode is jumpIfFalse (99%)"
		(next == JUMP_FALSE and: [ (rcvr < arg) not ]) ifTrue: [ 
			| offset |
			offset := ...
			^ self jump: offset ].
		"Cases for jump if true and other boolean values..."
		...
		"Not followed by jump, but integers nevertheless, avoid the send"
		^ self pushBoolean: rcvr < arg ].

	"Slow path: send"
	messageSelector := self specialSelector: 2.
	argumentCount := 1.
	self normalSend
```