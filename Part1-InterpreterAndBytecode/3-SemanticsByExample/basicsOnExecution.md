## Bytecode Semantics by Example
@cha:SemanticsByExample

This chapter explains how bytecode is executed by the virtual machine in a high-level manner, using examples.
The idea is to get you used to the stack semantics.
For this purpose, this chapter first approximates the concept of execution contexts.
Execution contexts represent the execution of methods: they hold the values of temporary variables and values pushed to the stack and they allow one to suspend the execution of methods by remembering the executed instructions.

The interpreter chapter will explain how these semantics map to the low(er)-level code of the interpreter and the compiler.
Later, contexts will come back as reflective reifications, allowing Pharo to support its powerful debugger!

### Setting up the Scene

Let us consider the two following methods, `Rectangle>>#width` and `Point>>#x`, with the following source code:

```
Point >> x
	^ x

Rectangle >> width
	| cornerX originX |
	cornerX := corner x.
	originX := origin x.
	^ cornerX - originX
```

Before being executable, this source code is translated to a format more _pleasant to the execution_.
Indeed, the Pharo bytecode compiler, outside of the scope of this book, translates this source code into a `CompiledMethod` object.
The compiled method object, as we studied before, contains a frame of literals and a set of bytecode instructions.

The method `Point>>#x` has the bytecode sequence 248, 8, 1, 0, and 92.

```
(Point >> #x) bytecode
  #[248 8 1 0 92]
```

To better understand what each of those bytes means, Pharo methods implement `symbolicBytecodes`, which returns a textual description of the method's bytecode sequence.
The next code snippet shows the textual representation of `Point>>#x`'s bytecode.
For each bytecode, it is first listed the bytecode index within the method, followed by the sequence of bytes that make up the instruction printed in hexa between angle brackets, and finally a textual description of that instruction (as shown in Listing *@symbolic@*).


```caption=Describing the byte codes of the accessor `x` of `Point`.&anchor=symbolic
(Point >> #x) symbolicBytecodes
	25 <F8 08 01> callPrimitive: 264
	28 <00> pushRcvr: 0
	29 <5C> returnTop
```

For method `Point>>#x`, the five bytes `248 8 1 0 92` do actually represent three different instructions.

- The first instruction, `<F8 08 01>`, is the instruction `callPrimitive` 264.
- The second instruction, `<00>`, is the instruction `pushRcvr` 0, which pushes the first instance variable (using a 0-base index) of the receiver (`self`).
- The final instruction, `<5C>`, is the instruction `returnTop`, which returns from the method with the value found in the top of the stack.



You may wonder why the first bytecode index does not start 1 or 0. This is because the bytecode sequence is part of a compiled method and the compiled method contains a memory region to hold some constants such as numbers, message selectors,... This region is called the literal frame. 

### Simplifying the Presentation

During this chapter we will, for pedagogical purposes, avoid explaining the call primitive bytecode.
Regarding this chapter, such a bytecode is just an optimization that will be explained in later chapters.
However, as with any optimization, the code will remain semantically valid without it, so we will consider that the previous method is as follows instead:

```
(Point >> #x) symbolicBytecodes
	28 <00> pushRcvr: 0
	29 <5C> returnTop
```

In addition, you may have noticed that textual bytecode descriptions use 0-based indexes to refer to instance variables and that the instruction names remain a bit cryptic.
This means that to understand an instruction we need to have in our mind the entire layout of a class and its hierarchy, in addition to all the potential acronyms and abbreviations.
To avoid this mental overhead, this chapter will show instead the resolved names and nicer reading descriptions.
We, however, invite the reader to read many method bytecodes to get used to the terminologies.
With these simplifications, we will show methods as follows:

```
(Point >> #x) symbolicBytecodes
	28 <00> push instance variable: x
	29 <5C> return top of stack
```

### Execution Model

Now that we have a grasp on how bytecodes are represented, we need to understand how execution is handled before getting into a concrete example.
This section introduces method contexts, or contexts for short, an abstraction that represents a method execution.
Method contexts model the state of the execution of a single method, and combined turn into a _call stack_.

#### Contexts

Executing a method requires that we:

1. have access to method arguments, and `self`.
2. remember the state of different local variables.
3. remember the state of expressions' subexpressions.
4. execute one by one all the instructions in the method.
5. remember the last executed instruction on a message send.
6. remember the caller method execution, to return to it.

All this is supported with a simple data structure, that we will call a _context_ that holds onto (as shown in Figure *@activation1@*):

- the method's program counter containing the next instruction to be executed,
- an array of temporary variables containing the values for each instance variable,
- a stack of values for intermediate expressions, and 
- a pointer to the caller's context, namely the _sender_, which contains the suspended execution of the caller method.

![A context to represent the execution of method Point>>#x. %width=65&anchor=activation1](figures/interpreter_activation.pdf)


Every time an instruction is executed, the program counter will advance to the next instruction.
An exception is _jump instructions_, which will make the program counter _jump_ to a specific program counter.
Moreover, storing the program counter allows one to save the execution of a method, and continue it later, for example when a message is sent.

Temporary variables are tracked in the temporary variable array.
The value stack stores the results of subexpressions, allowing an arbitrary number of nested expressions.

**About the figures:** Inspired by the Smalltalk-80 Blue Book {!citation|ref=Gold83a!}, we will show how methods are executed with figures similar to the next one.
Figure *@activation1@* shows the list of bytecodes in the method and the context object.
An arrow points to the next instruction to execute in the instruction list.



#### Understanding Bytecode Execution

Executing bytecode follows, inspired by how actual hardware works, a fetch-decode-execute cycle.
1. The byte at the program counter is fetched (read).
2. The read byte is decoded, and mapped to the instruction to execute.
3. Finally, the instruction is executed, and the program counter is set to the next instruction.




### Step by Step Method Execution by Example

![ A context ready to execute the method `Point>>#width`. %width=90&anchor=activation-step01](figures/interpreter_activation-step01.pdf)


Let's consider the following expression:

```
((1@2) corner: (3@4)) width
``` 

This expression creates a rectangle object, and sends it the message `width`, activating the method presented at the beginning of this chapter.

When the method `Rectangle>>#width` gets activated, a context is created for it.
This context is initialized as follows (as shown in Figure *@activation-step01@*):

- it references the method executed,
- its program counter points on the first bytecode of the method,
- it has one entry for each temporary variable, initialized to `nil`,
- it starts with an empty stack, and 
- its sender is the context sending the `((1@2) corner: (3@4)) width` message, avoided in the example for simplicity.

![After executing the first instruction, the context stack contains a reference to the `corner` instance variable (e.g. `3@4`).%width=90&anchor=activation-step02](figures/interpreter_activation-step02.pdf)

### Step 1: Pushing Values to the Stack

The first instruction in the method `Rectangle>>#width` pushes to the stack the value of the `corner` instance variable.
After its execution, the stack contains a new element: a reference to the object `3@4` (as shown in Figure *@activation-step02@*).
Moreover, the program counter increases, indicating that the next instruction to execute is a message send.


![ A new context is created for the execution of message `x` sent to the object popped of the stack of `Rectangle>>#width` context. The executing method is `Point>>#x`. `Rectangle>>#width` instruction pointer refers to the next instruction to execute once `x` message will return.%anchor=activation-step03&width=90](figures/interpreter_activation-step03.pdf)


### Step 2: Message Sends

The current instruction to execute, at program counter 26, performs the message send `x`.
To execute a message send:

- The execution first finds the receiver and looks up the method to execute. The receiver and arguments of the message are the top elements in the stack. In our case, since `x` is a message without argument, the receiver is the top of the stack, our `3@4` object. Thus, looking up the selector `x` from the class `Point`, yields the method `Point>>#x`.

- Then, the instruction pops the receiver (and arguments that we do not have here) from the stack and creates a new method context. As shown in Figure *@activation-step03@*, the method context refers to the method to execute (here `Point>>#x`) and the message receiver.

- The interpreter initializes the new context so that its program counter points to the first bytecode of the method (28). It initializes all temporaries to `nil` (none in this case), starts with an empty stack, and sets as sender the previous execution context.

- At this point, the executing method is `Point>>#x` with the receiver `3@4`.

### Steps 3 and 4: Push and Return

The first bytecode in the method `Point>>#x` pushes the value of the instance variable `x` of `3@4` to _current_ context's stack (as shown in Figure *@activation-step04@*).

![Starting the execution of  `Point>>#x` -- pushing the value of `x` to the stack. %anchor=activation-step04&width=90](figures/interpreter_activation-step04.pdf)

The last instruction in this method returns the control to the sender context.
The return value of this instruction is the top of the stack (3).
When the return instruction gets executed, the current execution context gets discarded, and its sender becomes (again) the current execution context.
The return value is then pushed to the stack of the new current context as shown in Figure *@activation-step05@*.

![When the method `Point>>#x` returns a value - it is pushed to the caller context stack, the context representing its execution is discarded, and the execution resumes on the previous context.  %anchor=activation-step05&width=90](figures/interpreter_activation-step05.pdf)

Now we are back executing our method `Rectangle>>#width`, and we are ready to restart the execution from where it was suspended: the program counter 27.


### Step 5: Popping and Storing into Temporary Variables

Back in `Rectangle>>#width`, the next instruction pops the top of the stack (oh this is the return value of the `x` message send!) and stores it in the temporary variable `cornerX` as shown in Figure *@activation-step06@*. Notice that this instruction stores the value and pops it to the stack all at once.
Other instructions allow one to do stores without pops, or just pops without stores, or even pop combined with other instructions.
Such instructions will be explained in the chapter about bytecode and interpreter optimizations.

![The result of the message `x` is stored in the temporary `cornerX`. %anchor=activation-step06&width=90](figures/interpreter_activation-step06.pdf)


### Steps 6 through 12: Repeat with the Rectangle's origin

The next steps are similar to the previous ones, but done with the `origin` instance variable.
The method pushes the `origin` instance variable value (1@2) to the stack, sends it the message `x`, and when it returns it pops the result and stores it in the temporary variable `originX`.

![ Before pushing the temporaries to the stack before sending the message `-`.%anchor=activation-step11&width=90](figures/interpreter_activation-step11.pdf)

Now we are ready for the last part of this method: the subtraction!
But before executing the subtraction, we need to push all of its operands to the stack as shown in Figure *@activation-step11@*.
Instructions at program counters 31 and 32 push the values of the temporary variables and leave us with the following context as shown in Figure *@activation-step1213@*. It is important to see that when we execute `3 - 1` the receiver, (here the object `3`) is not on top of the stack but below the argument (here the object `1`).

![Before sending the message `-`. Remark that the receiver is not the top of the stack. %anchor=activation-step1213&width=90](figures/interpreter_activation-step1213.pdf)

### Step 13: The Subtraction Primitive

At program counter 33 we perform the message send `-`.
Again, the send finds the receiver and looks up the method to execute.
Here, the subtraction is a binary selector with one argument, thus the receiver is the value just before the top of the stack, the small integer 3 as shown in Figure *@activation-step1213@*.
Thus, looking up the selector `-` from the class `SmallInteger`, yields the method `SmallInteger>>-`.

Differently from the previous methods, `SmallInteger>>#-` is a primitive method:

```
SmallInteger >> - aNumber
	<primitive: 2>
	^super - aNumber
```

Primitive methods work differently from normal methods: they execute directly on the caller context (our `Rectangle>>#width` context).
If the primitive instruction succeeds, the primitive operands are popped and the result is pushed.
This is what happens in this case, `3 - 1` is a properly working subtraction that results in the value 2 as shown in Figure *@activation-step14@*.

![ The primitive succeeds - it did not create a new context but worked directly in the context of `Point>>#width`.%anchor=activation-step14&width=90](figures/interpreter_activation-step14.pdf)

If the primitive instruction fails, we proceed to execute the method normally.
The instruction pops the receiver (and arguments that we did not have here) from the stack, and creates a new method context.
The method context will reference the method to execute to `Point>>#-`, the message receiver, initialize its program counter to the first instruction that will execute `super - aNumber`.

### Points to consider

The execution of the method `Point>>#width` illustrates some aspects that a real system will have to address. 

- First the context stack may grow more depending on the level of instructions nesting. When using a contiguous memory region to represent compiled methods (as opposed to use a pointer to a separate data structure) the maximum depth size of the stack should be computed to avoid overflow.

- Second, contexts are a nice abstraction to understand program execution and to represent the execution  for reflective operations but they have drawbacks and because of this the virtual machine uses a different representation called _stack frame_ for normal execution and use contexts for reflective operation such as `thisContext`.

We list here the limits of the context representation:

  - Each time a method sends a message, the receiver and arguments have to be copied to the new context. This is clearly a lost of time. 
  - The representation of contexts as objects (also called a spaghetti stack), can spread stack elements in different memory location breaking the locality of data that microprocessors use to speed up execution.
  
Representing execution elements in contiguous way improves this two drawbacks. 


### Conclusion

This chapter presented a high-level overview of bytecode execution:
- bytecode is executed one after the other in a method.
- jump instructions change the sequential aspect of execution and move the program counter to an arbitrary instruction.
- execution state is stored in contexts containing a stack and temporary variables. Values are stored into temporary variables using store instructions, and the results of subexpressions are stored in the context stack.
- contexts also store the program counter, which is useful for following the execution and suspending a method execution
- a message send suspends the current context and creates a new context for the called method
- primitive methods execute a primitive instruction on the context of the sender, without creating a context. Only if the primitive fails a context is created and the fallback bytecode is executed.
- from the perspective of the _sender_ context, the execution of a message send is a black box: a message send pops the arguments, and pushes the results. The sender does not need to know the implementation of the called method, whether it is a primitive method or not.


### Little todo

- explain briefly why bytecode starts a 25 or so.
- 
