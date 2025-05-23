## Methods, Bytecode, and Primitives
@cha:MethodBytecode

This chapter explains the basics of Pharo execution: methods and how they are represented internally.
Methods execute one after the other, and _call_ each other by means of message-send operations.
Under the hood, methods execute using a stack machine.
A stack holds the current calls and their values.
Bytecode instructions and primitives manipulate this stack with push and pop operations.

In this chapter, we explain in detail how methods are modeled using the Sista bytecode set {!citation|ref=Bera14a!}.
We explain how bytecode and primitive instructions are executed using a conceptual stack.
The bytecode interpreter and the call stack are introduced in Chapter *@cha:Slang@*.

### Intermezzo: Variables in Pharo

To understand how Pharo code works, it is useful to do a quick reminder on how variables work.
In this chapter, we will deal with the low-level representation of variables (how read/writes are implemented).
For a more complete overview on variables and their usage, please refer to Pharo by Example {!citation|ref=Duca17a!}.

Pharo supports three main kinds of variables.
Each kind of variable is stored differently in memory and has a different lifetime and behavior i.e., they are allocated and deallocated at different moments in time.

**`self` and `super`:** `self` and `super` are so called _pseudo variables_, i.e., syntactically looking like variables but defined by the compiler and with special semantics. Differently from normal variables, `self` and `super` cannot be manually assigned. Both `self` and `super` are defined automatically by the compiler, and bound when a method starts executing. Both these pseudo variables have as value the current message receiver: the object that received the message that led to the current method execution.

The key difference between `self` and `super` is how they change the method lookup semantics in a message send.
Messages to `self` (e.g., `self foo`) perform a traditional method lookup starting from the receiver.
Messages to `super` (e.g., `super foo`) start the method lookup algorithm from the superclass of the class that defines the current method.
We will review the method lookup semantics when looking at the interpreter implementation in the following chapters.

**Temporary variables and arguments:** A method has a list of formal arguments and a list of manually declared temporary variables.
Temporary variables and arguments are only accessible within the method that defines them and live during the entire method's execution.
In other words, they are allocated when a method execution starts and deallocated when a method returns.
Each method invocation has its own set of temporary variables and arguments, properly allowing recursion and concurrency.

Temporary variables and arguments have each a unique 0-based index per method.
Moreover, they share the same index namespace, meaning that no two temporaries or arguments can have the same index.
For example, assuming a method with `N` arguments and `M` temporary variables, its arguments are indexed from 0 to `N`, and its temporary variables are indexed from `N+1` to `N+M`.

**Instance variables:** Instance variables are variables declared in a class, and allocated on each of its instances.
That is, each instance has its own set of the instance variables declared in its class, and can directly access only its own instance variables.
Instance variables live as long as their containing object.
Instance variables are allocated as part of an object's slots, occupying a reference slot, and are deallocated as soon as an object is garbage collected.

Instance variables have also a unique 0-base index per ascending hierarchy, because an instance contains all the instance variables declared in its class and all its superclasses.
For example, given the class `Foo` declaring `N` instance variables and having a superclass `Bar` declaring `M` instance variables, the variables declared in `Bar` have indexes from 0 to `M`, the variables declared in `Foo` have indexes from `M+1` to `M+N`.

**Literal variables:** Literal variables are variables declared either as _Shared Variables_, _Shared Pools_ or _Global Variables_.
These variables have a larger visibility than the two other kind of variables.
Class variables are visible by all classes and instances of a hierarchy.
Shared Pools work like shared variables but can be imported in different hierarchies.
Global variables are globally visible.
Literal variables live as long as the program, or until a developer decides to explicitly undeclare them.

Literal variables do not have an index: they are represented as an association (a key-value object) and stored in dictionaries.
Methods using literal variables store the corresponding associations in their literal frame, as described in Chapter *@cha:ObjectRepresentation@*.


### Stack Bytecode

Pharo encodes bytecode instructions using the Sista bytecode set {!citation|ref=Bera14a!}.
The Sista bytecode set defines a set of stack instructions with instructions that are one, two or three bytes long.
Instructions fall into five main categories: pushes, stores, sends, returns, and jumps.

This section gives a general description of each bytecode category.
Later we present the different optimizations, the bytecode extensions and the detailed bytecode set.

#### A Stack Machine

Pharo bytecode works by manipulating a stack, as opposed to registers.
Typically, an operation accesses its operands from the stack, operates on them, and places the result back on the stack.
We will call this stack the **value stack**, also known as the **operand stack**, to differentiate it from the call stack that will be studied in a later chapter.

For example, Listing *@push27@* shows the three stack instructions required to evaluate the expression `2+7`.
First, two push instructions push the values 2 and 7 to the value stack.
Second, the `+` instruction pops the two top values of the stack, operates on them producing the number 9, and finally pushes that value to the value stack.

```caption=Pseudo-bytecode performing 2+7.&anchor=push27
push 2
push 7
+
```

#### Push Instructions (to the value stack)

Push instructions are a family of instructions that read a value and add it to the top of the value stack.
Different push instructions are:

- push the current method receiver (`self`)
- push an instance variable of the receiver
- push a temporary/argument
- push a literal
- push the value of a literal variable
- push the top of the stack, duplicating the stack top

#### Store Instructions (to a variable)

Store instructions are a family of instructions that write the value at the top of the value stack to a variable.
Different store instructions are:

- store into an instance variable of the receiver
- store into a temporary variable
- store into a literal variable


#### Control Flow Instructions: Jumps

Control flow instructions are a family of instructions that change the sequential order in which instructions naturally execute.
In most programming languages, such instructions represent conditionals, loops, case statements.
In Pharo, all these boil down to jump instructions, a.k.a., gotos.

Different jump instructions are:

- Conditional jumps pop the top of the value stack and transfer the control flow to the target instruction if the value is either `true` or `false`.
- Unconditional jumps transfer the control flow to the target instruction regardless of the values on the value stack.

#### Send and Return Instructions

Send instructions are a family of instructions that perform a message send, activating a new method on the call stack.
Send instructions need a selector and its number of arguments, and will conceptually work as follows:
- pop receiver and arguments from the value stack
- lookup the method to execute using the receiver and message selector
- execute the looked-up method
- push the result to the top of the stack

Conversely, to send instructions, return instructions are a family of instructions that return control to the caller method, providing the return value to be pushed to the caller's value stack.


### Primitive Methods

Some operations such as integer arithmetics or bitwise manipulations cannot be expressed by means of message sends and methods.
Pharo expresses such operations through primitives: low-level functions implementing essential or optimized operations.

Primitives are exposed to Pharo through primitive _methods_.
A primitive method is a bytecode method that has a reference to a primitive function.
For example in Listing *@plus@*, the method `SmallInteger>>#+` defining the addition of immediate integers is marked to as primitive number 1.

```caption=The SmallInteger addition method is a primitive method.&anchor=plus
SmallInteger >> + addend
	<primitive: 1>
	^super + addend
```

Primitives are implemented as stack operations having only access to the value stack.
When a primitive function is executed, the value stack contains the method arguments.

A key difference between primitive functions and bytecode instructions is that primitives _can fail_.
When a primitive method is executed, it starts with executing the primitive function.
If the primitive function succeeds, the primitive method returns the result of the primitive function.
If the primitive function fails, the primitive method _falls back_ to the method's bytecode.
For example, in the case above, if the primitive 1 fails, the statement `^ super + addend` will be executed.

**Design note: fast vs slow paths.** The failure mechanism of primitives is generally used to separate fast paths from slow paths.
For example, integer addition has a very compact and fast implementation if we assume that both operands are immediate integers.
However, by design Pharo needs to support the arithmetic between different types such as immediate integers, boxed large integers, floats, fractions, scaled decimals.
In this scenario, a primitive is used to cover the fast and common case: adding two immediate integers.
The primitive performs a runtime type check: it verifies that both operands are immediate integers.
If the check succeeds, the primitive performs its operation and returns without executing the fallback bytecode.
This first execution path is the _fast path_.
If the check fails, the primitive fails and the method's fallback bytecode implements the slower type conversion for the other type combinations. That is the _slow path_.


### Conclusion

In this chapter we studied how the Pharo virtual machine represents code.
- The Pharo VM defines a stack machine: computation is expressed by manipulating a stack with push and pop operations. These operations are called bytecode instructions and primitive instructions.
- Code is organized in methods. Methods contain at most one primitive, a sequence of bytecode instructions and a list of literal values.
- Bytecode instructions manipulating variables carry semantic information about them: there are special instructions for instance variables, shared variables and temporary variables. This allows the VM to concentrate on the execution and not to do name analysis at runtime to guess what kind of variable is each name.
- Primitive instructions can fail. If they succeed in their computation, they pop their arguments and push the result. If they fail, they leave the value stack untouched and return an error code.
- Primitive methods make a strong distinction between slow and fast paths: in the fast path they execute the primitive instruction. If it was a success, execution continues in the sender. Otherwise, the method's bytecode is executed.