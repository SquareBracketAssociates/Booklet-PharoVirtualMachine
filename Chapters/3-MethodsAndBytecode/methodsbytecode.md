## Methods, Bytecode and Primitives


This chapter explains the basics of Pharo execution: methods and how they are internally represented.
Methods execute one after the other, and _call_ each other by means of message send operations.
Methods execute under the hood using a stack machine.
A stack holds the current calls and their values.
Bytecode instructions and primitives manipulate this stack with push and pop operations.

In this chapter we explain in detail how methods are modelled using the sista bytecode set{!citation|ref=Bera14a!}.
We explain how bytecode and primitive instructions are executed using a conceptual stack.
The bytecode interpreter and the call stack are introduced in a following chapter.

### Compiled Methods

Pharo users write methods in Pharo syntax.
However, Pharo source code is just text and is not executable.
Before executing those methods, a bytecode compiler processes the source code and translates it to an executable form by performing a sequence of transformations until it generates a `CompiledMethod` instance.
A parsing step translates the source code into a tree data structure called an _abstract syntax tree_, a name resolution step attaches semantic to identifiers, a lowering step creates a control flow graph representation, and finally a code generation step produces the `CompiledMethod` object containing the code and meta-data required for execution.

#### Intermezzo: Variables in Pharo

To understand how Pharo code works, it is useful to do a quick reminder on how do variables work.
In this chapter we will deal with the low-level representation of variables (how read/writes are implemented).
For a more complete overview on variables and their usage, please refer yourselves to Pharo by Example{!citation|ref=Duca17a!}.

Pharo supports three main kind of variables.
Each kind of variable is stored differently in memory and has different lifetime and behavior _i.e.,_ they are allocated and deallocated at different moments in time.

**Temporary variables and parameters:** A method has a list of formal parameters and a list of manually declared temporary variables.
Temporary variables and parameters are only accessible within the method that defines them and live during the entire method's execution.
In other words, they are allocated when a method execution starts and deallocated when a method returns.
Each method invocation has its own set of temporary variables and parameters, property allowing recursion and concurrency (understanding why is left as an exercise for the reader).

Temporary variables and parameters have each a unique 0-based index per method.
Moreover, they share the same index namespace, meaning that no two temporaries or parameters can have the same index.
For example, assuming a method with `N` parameters and `M` temporary variables, its parameters are indexed from 0 to `N`, and its temporary variables are indexed from `N+1` to `N+M`.

**Instance variables:** Instance variables are variables declared in a class, and allocated on each of its instances.
That is, each instance has its own set (or copy) of the instance variables declared in its class, and can directly access only its own instance variables.
Instance variables live as long as its containing object.
Instance variables are allocated as part of an object slots, occupying a reference slot, and are deallocated as soon as an object is garbage collected.

Instance variables have also a unique 0-base index per ascending hierarchy, because an instance contains all the instance variables declared in its class and all its superclasses.
For example, given the class `Class` declaring `N` instance variables and having a superclass `Super` declaring `M` instance variables, the variables declared in `Super` have indexes from 0 to `M`, the variables declared in `Class` have indexes from `M+1` to `M+N`.

**Literal variables:** Literal variables are variables declared either as _Class Variables_, _Shared Variables_ or _Global Variables_.
These variables have a larger visibility than the two other kind of variables.
Class variables are visible by all classes and instances of a hierarchy.
Shared variables work like class variables but can be imported in different hierarchies.
Global variables are globally visible.
Literal variables live as long as the program, or a developer decides to explicitly undeclare them.

Literal variables do not have an index: they are represented as an association (a key-value object) and stored in dictionaries.
Methods using literal variables store the corresponding associations in their literal frame, as we will see next.

#### Literals and the Literal Frame

Pharo code includes all sort of literal values that need to be known and accessed at runtime.
For example, the code below shows a method using integers, arrays and strings.

```caption=Source code with several literals
MyClass >> exampleMethod
    self someComputation > 1
		ifTrue: [ ^ #() ]
		ifFalse: [ self error: 'Unexpected!' ]
```

In Pharo, literal values are stored each in different a reference slot in a method.
The collection of reference slots in a method is called the _literal frame_.
Remember from the object representation chapter, that `CompiledMethod` instances are variable objects that contain a variable reference part and a variable byte indexable part.

Commonly, the literal frame contains references to numbers, characters, strings, arrays and symbols used in a method.
In addition, when referencing globals (and thus classes), class variables and shared variables, the literal frame references their corresponding associations.
Finally, the literal frame references also runtime meta-data such as flags or message selectors required to perform message-sends.

It is worth noticing that the literal frame poses no actual limitation to what type of object it references.
Such capability is exploited in rare cases when a method's behavior cannot be expressed in Pharo syntax.
This is for eample the case of foreign function interface methods that are compiled by a separate compiler and stores foreign function meta-data as literals.

#### Method Header

All methods contain at least one literal named the _method header_, referencing an immediate integer representing a mask of flags.

- **Encoder:** a bit indicating if the method uses an alternate bytecode set.
- **Primitive:** a bit indicating if the method has a primitive operation or not.
- **Number of parameters:** 4 bits representing the number of parameters of the method.
- **Number of temporaries:** 6 bits representing the number of temporary variables declared in the method.
- **Number of literals:** 15 bits representing the number of literals contained in the method.
- **Frame size:** 1 bit representing if the method will require small or large frame sizes.

The encoder and primitive flags will be covered later in this chapter.
The frame size will be explored in the context reification chapter.

#### Method Trailer

Following the method literals, a `CompiledMethod` instance contains a byte-indexable variable part, containing bytecode instructions.
However, it is of common usage in Pharo to make this byte-indexable part slightly larger to contain trailing meta-data after a method's bytecode.
Such meta-data is called the _method trailer_.

The method trailer can be arbitrarily long, encoding binary data such as integers or encoded text.
Pharo usually uses the trailer to encode the offset of the method source code in a file.
It has, however, also been used to encode a method source code in utf8 encoding, or zipped.

### Stack Bytecode and the Sista Bytecode Set Overview

Pharo encodes bytecode instructions using the Sista bytecode set{!citation|ref=Bera14a!}.
The Sista bytecode set defines a set of stack instructions with instructions that are one, two or three bytes long.
Instructions fall into five main categories: pushes, stores, sends, returns, and jumps.

This section gives a general description of each bytecode category.
Later we present the different optimizations, the bytecode extensions and the detailed bytecode set.

#### A Stack Machine

Pharo bytecode works by manipulating a stack, as opposed to registers.
Typically, an operation accesss its operands from the stack, operates on them, and places the result back on the stack.
We will call this stack the value stack or operand stack, to differentiate it from the call stack that will be studied in a later chapter.

For example, the following code shows the tree stack instructions required to evaluate the expression `2+7`.
First, two push instructions push the values 2 and 7 to the stack.
Second, the `+` instruction pops the two top values in the stack, operates on them producing the number 9, and finally pushes that value to the stack.

```caption=Pseudo-bytecode performing 2+7
push 2
push 7
+
```

#### Push Instructions

Push instructions are a family of instructions that read a value and add it to the top of the value stack.
Different push instructions are:

- push the current method receiver (`self`)
- push an instance variable of the receiver
- push a temporary/parameter
- push a literal
- push the value of a literal variable
- push the top of the stack, duplicating the stack top

#### Store Instructions

Store instructions are a family of instructions that write the value on the top of the stack to a variable.
Different store instructions are:

- store into an instance variables of the receiver
- store into a temporary variable
- store into a literal variable

#### Control Flow Instructions -- Send and Return

Send instructions are a family of instructions that perform a message send, activating a new method on the call stack.
Send instructions are annotated with the selector and number of arguments, and will conceptually work as follow:
- pop receiver and arguments from the value stack
- lookup the method to execute using the receiver and message selector
- execute the looked-up method
- push the result to the top of the stack

Conversely to send instructions, return instructions are a family of instructions that return control to the caller method, providing the return value to be pushed to the caller's value stack.

#### Control Flow Instructions -- Jumps

Control flow instructions are a family of instructions that change the sequential order in which instructions naturally execute.
Different jump instructions are:
- conditional jumps pop the top of the value stack and transfer the control flow to the target instruction if the value is either `true` or `false`
- unconditional jumps transfer the control flow to the target instruction regardless of the values on the value stack
	
### Primitive Methods

Some operations such as integer arithmetics or bitwise manipulation cannot be expressed by means of message sends and methods.
Pharo express such operations through primitives: low-level functions implementing essential or optimized operations.

Primitives are exposed to Pharo through primitive _methods_.
A primitive method is a bytecode method that has a reference to a primitive function.
For example, the method `SmallInteger>>#+` defining the addition of immediate integers is marked to as primitive number 1.

```caption=The SmallInteger addition method is a primitive method
SmallInteger >> + addend
	<primitive: 1>
	^super + addend
```

Primitives are implemented as stack operations having only access to the value stack.
When a primitive function is executed, the value stack contains the method arguments.

A key difference between primitive functions and bytecode instructions is that primitives _can fail_.
When a primitive method is executed, it executes first the primitive function.
If the primitive function succeeds, the primitive method returns the result of the primitive function.
If the primitive funciton fails, the primitive method executes _falls back_ to the method's bytecode.
For example, in the case above, if the primitive 1 fails, the statement `^ super + addend` will get executed.

**Design Note: fast vs slow paths.** The failure mechanism of primitives is generally used to separate fast paths from slow paths.
For example, integer addition has a very compact and fast implementation if we assume that both operands are immediate integers.
However, Pharo by design needs to support the arithmetics between different types such as immediate integers, boxed large integers, floats, fractions, scaled decimals.
In this scenario, a primitive is used to cover the fast and common case: adding up two immediate integers.
The primitive performs a runtime type check: it verifies that both operands are immediate integers.
If the check succeeds, the primitive performs its operation and returns without executing the fallback bytecode.
This first execution path is the _fast path_.
If the check fails, the primitive fails and the method's fallback bytecode implements the slower type conversion for the other type combinations.


### Bytecode Encoding

### Bytecode Optimization

#### Optimising for Common Bytecodes

- push inst var 1 as a single bytecode
- super instructions (store and pop)

#### Optimising for Common Messages


	
The categories of objects that can be referred to directly by bytecodes are:
- the receiver and arguments of the message
- the values of the receiver's instance variables
- the values of any temporary variables required by the method
- special constants \(true, false, nil, -1, 0, 1, and 2\)
- special message selectors


```
BytecodeEncoder specialSelectors	
>>> #(#+ #- #< #> #'<=' #'>=' #= #'~=' #* #/ #'\\' #@ #bitShift: #'//' #bitAnd: #bitOr: #at: #at:put: #size #next #nextPut: #atEnd #'==' nil "class" 
#'~~' #value #value: #do: #new #new: #x #y)
```

### The Sista Bytecode

#### Bytecode Extensions

Some of the bytecodes take extensions. An extension is one or two bytes following the bytecode that further specify the instruction. An extension is not an instruction on its own, it is only a part of an instruction.
