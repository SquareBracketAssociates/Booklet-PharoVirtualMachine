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

### Global Lookup Cache

### Static Type Predictions

### Frameless Methods

### Instruction Lookaheads