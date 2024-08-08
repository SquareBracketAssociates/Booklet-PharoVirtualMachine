## Methods, Bytecode and Primitives

@@ Stef: I would cut the byte code (section 3.4) in a separate chapter. There is a big difference in information.


This chapter explains the basics of Pharo execution: methods and how they are internally represented.
Methods execute one after the other, and _call_ each other by means of message-send operations.
Methods execute under the hood using a stack machine.
A stack holds the current calls and their values.
Bytecode instructions and primitives manipulate this stack with push and pop operations.

In this chapter, we explain in detail how methods are modeled using the sista bytecode set{!citation|ref=Bera14a!}.
We explain how bytecode and primitive instructions are executed using a conceptual stack.
The bytecode interpreter and the call stack are introduced in Chapter *@cha:Slang@*.

### Compiled Methods

Pharo users write methods in Pharo syntax.
However, Pharo source code is just text and is not executable.
Before executing those methods, a bytecode compiler processes the source code and translates it to an executable form by performing a sequence of transformations until it generates a `CompiledMethod` instance.
A parsing step translates the source code into a tree data structure called an _abstract syntax tree_, a name resolution step attaches semantics to identifiers, a lowering step creates a control flow graph representation, and finally a code generation step produces the `CompiledMethod` object containing the code and meta-data required for execution.

#### Intermezzo: Variables in Pharo

To understand how Pharo code works, it is useful to do a quick reminder on how do variables work.
In this chapter, we will deal with the low-level representation of variables (how read/writes are implemented).
For a more complete overview on variables and their usage, please refer yourselves to Pharo by Example{!citation|ref=Duca17a!}.

Pharo supports three main kind of variables.
Each kind of variable is stored differently in memory and has a different lifetime and behavior _i.e.,_ they are allocated and deallocated at different moments in time.

**`self` and `super`:** `self` and `super` are so called _pseudo-variables_, _i.e.,_ syntactically looking variables but defined by the compiler and with special semantics. Differently from normal variables, `self` and `super` cannot be manually assigned. Both `self` and `super` are defined automatically by the compiler, and bound when a method starts executing. Both these pseudo-variables have as value the current message receiver: the object that received the message that led to the current method execution.

The key difference between `self` and `super` is how they change the method lookup semantics in a message-send.
Messages to `self` (_e.g.,_ `self foo`) perform a traditional method lookup starting from the receiver.
Messages to `super` (_e.g.,_ `super foo`) start the method lookup algorithm from the current method's superclass.
We will review the method lookup semantics when looking at the interpreter implementation in the following chapters.

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
For example, Listing *@exampleMethod@* shows a method using integers, arrays, and strings.

```caption=Source code with several literals&label=exampleMethod
MyClass >> exampleMethod
    self someComputation > 1
		ifTrue: [ ^ #() ]
		ifFalse: [ self error: 'Unexpected!' ]
```

In Pharo, literal values are stored each in different reference slot in a method.
The collection of reference slots in a method is called the _literal frame_.
Remember from the object representation chapter, that `CompiledMethod` instances are variable objects that contain a variable reference part and a variable byte indexable part.

- Commonly, the literal frame contains references to numbers, characters, strings, arrays and symbols used in a method.
- In addition, when referencing globals (and thus classes), class variables and shared variables, the literal frame references their corresponding associations.
- Finally, the literal frame references also runtime meta-data such as flags or message selectors required to perform message-sends.

It is worth noticing that the literal frame poses no actual limitation to what type of object it references.
Such capability is rarely exploited when a method's behavior cannot be expressed in Pharo syntax.
This is for example the case of foreign function interface methods that are compiled by a separate compiler and store foreign function meta-data as literals.

### Compiled Method

A compiled method is structured around a method header, a literal frame, a bytecode sequence and a method trailer as shown in Figure *@methodshape@*.

![.%anchor=methodshape](figures/compile_method_shape.pdf)

#### Method Header

All methods contain at least one literal named the _method header_, referencing an immediate integer representing a mask of flags.

- **Encoder:** a bit indicating if the method uses the default bytecode set or not.
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

### Stack Bytecode

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


#### Control Flow Instructions: Jumps

Control flow instructions are a family of instructions that change the sequential order in which instructions naturally execute.
In lots of programming languages, such instructions represent conditionals, loops, case statements.
In Pharo all these boil down to jump instructions, a.k.a., gotos.
Different jump instructions are:
- conditional jumps pop the top of the value stack and transfer the control flow to the target instruction if the value is either `true` or `false`
- unconditional jumps transfer the control flow to the target instruction regardless of the values on the value stack

#### Send and Return Instructions

Send instructions are a family of instructions that perform a message send, activating a new method on the call stack.
Send instructions are annotated with the selector and number of arguments, and will conceptually work as follow:
- pop receiver and arguments from the value stack
- lookup the method to execute using the receiver and message selector
- execute the looked-up method
- push the result to the top of the stack

Conversely to send instructions, return instructions are a family of instructions that return control to the caller method, providing the return value to be pushed to the caller's value stack.

	
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


### Bytecode Encoding and Optimizations

The instructions of a method are encoded as bytes, that need to be decoded to either interpret them, JIT compile them or decompile them.
Each instruction is made of an **opcode**, or operation identifier, followed by zero or more arguments.
For example, the instruction `push instance variable 42` is encoded with bytes `#[226 42]`, where 226 is the opcode identifying the  `push instance variable`, and the second byte (42) is the index of the instance variable to read.

#### Variable-length Bytecode Encoding

Pharo encodes bytecode instructions using a variable-length encoding: instructions are encoded using one, two or three bytes.
The encoding is designed for compactness and ease of interpretation.
Commonly used instructions are encoded with less bytes, rarely used instructions use more bytes.

The variable-length bytecode design has two consequences:
1. **Compact representation of bytecode methods.** Using shorter bytecode sequences for common instructions works as a compression mechanism. This allows the virtual machine to fetch less bytes during interpretation, and to use less space to encode methods.
2. **Ambiguity during decoding.** Bytecode in a method needs to be decoded for reasons such as debugging or decompilation. However, decoding cannot start from any arbitrary point in a variable-length encoding. Consider a two-byte bytecode. If we start decoding bytecode instructions from the second byte, the decoder will interpret this byte as the first byte of an instruction. In the best case, the decoder will eventually fail. In the worst case, the decoder succeeds and returns a wrong decoding.

In the following section we will go into how such optimizations take place concretely in the Pharo's bytecode set.

#### Optimising for Common Bytecode Instructions

As we said before, the variable-length bytecode encoding allows for shorter bytecode sequences for common instructions.
For example, we can take the most common bytecode from the Pharo12 release (build 1521) using the script that follows.
The script takes all the compiled code (methods and blocks), decodes all their instructions and groups them by their bytes.

```caption=Obtaining the most common bytecode instructions
((CompiledCode allSubInstances flatCollect: [ :e | e symbolicBytecodes ])
	groupedBy: [ :symBytecode | symBytecode bytes ])
		associations
			sorted: [ :a :b | a value size > b value size  ]
```

From that list, the 5 most common bytecode are:

| Instruction | Count |
| --- | --- |
| Pop | 209537 |
| Push self | 201567 |
| Push first temp/arg | 163965 |
| Send message in 1st literal | 77767 |
| Method return | 77090 |

We see in this list that the first three are largely more numerous than the two last.
This tendency continues in the entire list of bytecode following an exponential decay.
The first fifty instructions happen tenths of thousands of times, while the vast majority appear less than a thousand.

This observation is enough motivation to optimize such _very common_ cases.
Indeed, amongst the 255 most common instructions, 183 are already encoded as a 1 byte instruction.

#### Encoding of Single-byte instructions

Instructions such as `pop` or `push self` are single instructions that do not need any parameter.
The encoding of these instructions is straight forward: they are given a single byte.
For example, `pop` is encoded as 216, while `push self` is encoded as 76.

There are however other common instructions that have parameters.
This is the case, for example, of the `push instance variable` bytecode that is parameterized with the index of the reference slot in the receiver (the instance variable) to push.
To encode this instruction as a single byte, the index is encoded within the instruction.
That is, the bytecode `push instance variable at 1` is encoded as 0, the bytecode `push instance variable at 2` is encoded as 1.

Single-byte parametrized bytecode instructions are organized in ranges, often of a size that is a power of 2.
For example, 1-byte `push instance variable` instruction is organized in a range of 16 instructions (2^4).
1-byte `push instance variable` instructions are encoded with bytes from 0 to 15, parameterized with indexes from 1 to 16 respectively.

An alternative way of seeing this encoding is to see that an instruction opcode is not the byte on itself but the most significant bits of the byte. If we consider again the range of bytecodes `push instance variable`, the most significant nibble remains always zero regardless of the bytecode, while the lowest part always changes following the index to push.

```caption=Understanding encoding and nibbles
"The most significant nibble is always 0 for this range of bytecode"
0 to: 15 do: [ :e | self assert: ((e >> 4) bitAnd: 16rF) = 0 ].
"The least significant nibble is always the index to push"
0 to: 15 do: [ :e | self assert: (e bitAnd: 16rF) = e ]
```

#### Optimising for Common Bytecode Sequences

Besides common instructions, another useful observation is that many instructions are usually combined together.
Consider for example the statement `^ self`, which is commonly used to perform an early exit from a method, and inserted at the end of every method that does not have an explicit return.
A naïve translation of `^self` could use the following sequence of instructions.

```caption=A common bytecode sequence for returning self
push self
return top
```

Using two instructions requires -- at least -- two bytes, and forces the interpreter to pay twice the cost of instruction fetch/decode.
Pharo optimizes such common sequences using a single (often also single-byte) instruction to do the entire operation, often called (static) super instructions in the literature {!citation|ref=Pium98a!}.

#### Optimising for Common Messages and Literals

Another source of overhead happens on the over-reliance on literals.
In Pharo, each method has its own literal frame: literals and constants are not shared between methods, causing a potential redundancy and memory inefficiency.

One way to minimize such overhead is to design special instructions for well-known constants.
Constants such as `nil`, `true`, `false` need to be known by the VM for several tasks such as initializing instance variables, or interpret conditional jumps.
The VM benefits from this knowledge to provide specialized instructions such as `push true` that do not fetch the `true` object from the method literal frame but from the pool of constants known from the VM.

In the same venue, immediate objects can be crafted by the VM on the fly, avoiding the storage in the literal frame.
Instructions such as `push 0`, encoded as 80, represent the usage of constants that appear often, for example, in loops.
When executing those instructions, the VM create an immediate object by tagging a well-known value.

Finally, another variation of this optimization happens on common message sends _e.g.,_ arithmetic and comparisons selectors.
These selectors happen so often, that instead of storing the selector in the method's literal frame, they are stored in a global table of selectors called `special selectors`.
The Pharo bytecode set defines `send special selector` instructions.

### The Sista Bytecode Set

Pharo uses a bytecode set named Sista {!citation|ref=Bera14a!}.
The Sista bytecode set is a stack-based bytecode using common optimization as described before, including support for prefix instructions, and designed to be extensible with unsafe instructions.

This section explains some particularities of the bytecode set and finishes with a table showing the bytecodes, their encoding and summarizing their optimizations.

#### Bytecode and Variable Indexes are 0-based, Primitives 1-based

While the Pharo programming language designs its indexed accesses with 1-base offsets, this is not the case of the underlying implementation.
Actually, it is only the primitives that start with a 1-based index.
In contrast:

- bytecode instructions are encoded in a 0-based fashion, making the value `0` a valid encoded instruction.
- all variables, temporaries and instance, use 0-based indexing. Thus, the bytecode to read the first instance variable is `push instance variable 0`. Similarly, the bytecode to read the first temporary variable is `push temporary variable 0`.


#### Temporary Variables vs Arguments

The Sista bytecode set inherits, mostly for historical reasons, several traits from previous the bytecode design.
One particularly interesting trait is that method arguments are modelled as the first (read-only) temporary variables in a method.
For example, while the method that follows has syntactically one argument and one temporary variable, the underlying implementation will have two temporary variables, from which the first is an argument.

```caption=Arguments are the first temporaries in a method.
MyClass >> methodWithOneArgAndOneTemp: arg

	| temp |
	...

(MyClass >> #methodWithOneArgAndOneTemp: ) numArgs. "1"
(MyClass >> #methodWithOneArgAndOneTemp: ) numTemps. "2"
```

This decision impacts the bytecode design in different ways.

1. First, to get the real number of temporaries in a method we need to substract the number of arguments from it.

```caption=Obtaining the real number of temporaries from a method.
realNumberOfTemporaries := aMethod numTemps - aMethod numArgs
```

2. We need to know the arguments of a method to index its temporaries. For example, reading the nth real temporary variable in a method, we need to read the temporary at offset `numArgs + nth - 1`~(remember that we need to substract 1 because variable indexes are 0-based).

#### Bytecode Extension Prefixes

Some bytecode instructions are limited by the encoding: for example, 2-byte instructions usually use one byte as opcode and one byte as argument, limiting the argument to a maximum of 255 values.
For example, the code that follows illustrates the code of the `long jump if false`, that jumps to a given target bytecode if a false is found in the stack. This 2-byte bytecode uses the second bytecode as a relative offset from the current bytecode.
Such a restriction can be too limiting for some applications. In the case of our example, this forbids us from having jumps longer than 255 bytes.

```caption=Sketching the jump if false
extJumpIfFalse
	| byte offset |

	byte := self fetchByte.
	offset := byte.
	self jumplfFalseBy: offset
```

To solve this issue, the sista bytecode includes two prefix instructions: `extension A` and `extension B`.
Prefix instructions prefix normal instructions and work as meta-data for the following instruction: the semantics of a prefix depends on each instruction. Also, instructions that use an instruction _consume it_, zeroing its value for subsequent instructions.
The actual implementation of the `long jump if false` bytecode adds uses the value of it's prefix (if any) to reach further jump offsets.

```caption=Sketching the jump if false with extensions
extJumpIfFalse
	byte := self fetchByte.
	offset := byte + (extB << 8).
	numExtB := extB := extA := 0.
	self jumplfFalseBy: offset
```

The example that follows show two bytecode sequences, where the first jumps forward by 255 bytes while the second one jumps forward by 256 bytes.
The first sequence does not require any extensions, while the second one uses an extension.
Notice that the `long jump if false` computes it's offset as `byte + (extB << 8)`.
Thus, to compute an offset of 256, we should have an extension of value `1`, and a jump argument of value `0`.

```caption=Jumping more than 255 bytes
"Jump forward 255 bytes"
extJumpIfFalse 255

"Jump forward 256 bytes"
extA 1
extJumpIfFalse 0
```

In the case before, the prefix allows jumps to reach jump targets up to `65535` (`255 + (255 << 8)`).
To support larger values, prefixes can be composed: an instruction can have many prefixes that cummulate.
Following is the definition of the  `extension A` bytecode, which takes the previous value of the extension A, shifts it 8 bits to the left and adds it to the given value.

```caption=The extension implementation
extABytecode
	extA := (extA bitShift: 8) + self fetchByte.
	self fetchNextBytecode
```

Extensions are composed by adding many prefixes to a given instruction.
For example, the following example shows a jump with two extensions of 1 and 2, to a jump with argument 3.
This computes the jump offset of 66051 with the formula `(((1 << 8) + 2) << 8 + 3)`.

````caption=Combining extensions to jump above 65535 bytes
"Jump forward 66051 bytes"
extA 1
extA 2
extJumpIfFalse 3
```

#### Super sends

`super` sends deserve a special section for their own, specially because the Sista Bytecode set introduced _directed_ super sends in addition to the traditional ones.
When using the `super` keyword in Pharo, a message send is issued starting the method-lookup from the superclass of the current method's class instead of the receiver's class.
This means that to perform a super-send lookup, we need to access the current method's superclass, which should be encoded in the method or bytecode in some way.
Pharo's bytecode set allows for encoding such information in two ways:

- **Per-method encoded super sends:** Traditionally, each Pharo method contain as last literal a reference to it's class binding. When performing a normal super send, the algorithm fetches this last literal, then it's superclass, and starts the lookup algorithm from there.
- **Per-call-site encoded super sends:** _Directed super sends_ allow to specify as a stack argument the class from where to start the lookup. Directed super sends allow to specify different lookup classes per call-site, by pushing the lookup-class as the last element on the stack. Although initially meant for super sends, this bytecode can be used to control message sends per-call-site at the expense of larger bytecode and literal frames.

```caption=A directed send bytecode sequence
"This will lookup #some:message: starting from ClassToLookupFrom, using the receiver and argument found in the stack"
push receiver
push arg0
push ClassToLookupFrom
directed-send #some:message:
```

### Sista Bytecode Design Overview

It is not the goal of this book to give a formal definition of the bytecode, or to establish a standard carved in stone.
Instead, this section describes how the optimizations described above are used in the context of Pharo's bytecode, to understand the design space and how it is concretized.

The Sista bytecode defines a variable byte encoding using many of the encoding optimizations described above.
Not all bytes in those ranges are used, which allows potential extensions in the future.
Most of the single-byte bytecode, if not all, are optimized versions of more general instructions found in the range 224-255.

- Bytecodes 0-223 are single-byte bytecode
- Bytecodes 224-247 are two-byte bytecode
- Bytecodes 248-255 are three-byte bytecode

In this section, we will refer to `byte0` as the first byte of a instruction, `byte1` as the second byte (if exists) and `byte2` as the third byte (if exists).

#### Sista Optimized Bytecode

The range 0-75 encodes four diferent families of very common instructions: push receiver instance variable, push literal variable, push literal constant and push temporary variable.
Each family has many versions specialized for one particular argument, encoded as part of the instruction.

| Bytes | Description | Arguments |
| --- | --- | --- |
| 0-15 | Push receiver's instance variable at offset `x` | x = byte0 |
| 16-31 | Push literal variable value at offset `x` | x = byte0 && 0x1F |
| 32-63 | Push literal constant at offset `x` | x = byte0 && 0x1F|
| 64-75 | Push temporary variable at offset `x` | x = byte0 &&0xF |

The range 76-94 encodes push and return instructions with well-known constants, reducing the size of the literal frame.

| Bytes | Description | Arguments |
| --- | --- | --- |
| 76 | Push receiver | - |
| 77 | Push `true` | - |
| 78 | Push `false` | - |
| 79 | Push `nil` | - |
| 80 | Push `0` | - |
| 81 | Push `1` | - |
| 88 | Return receiver from method | - |
| 89 | Return `true` from method | - |
| 90 | Return `false` from method | - |
| 91 | Return `nil` from method | - |
| 92 | Return top of the stack from method | - |
| 93 | Return `nil` from block | - |
| 94 | Return top of the stack from block | - |

The range 96-127 encodes special sends: sends whose selector and number of arguments is well-known to the VM.
Such sends include common selectors such as arithmetics (_e.g.,_ `#+`, `#*`) or comparisons  (_e.g.,_ `#>`, `#==`).
This advantages of this encoding are two-fold.
First, it makes methods smaller as it requires less space in the literal frame
Second, the mapping allows the VM to make semantic optimizations.
For example, knowing that a particular send is an addition `#+` allows the VM to avoid message sends and generate specific code for it.

| Bytes | Description | Arguments |
| --- | --- | --- |
| 96-127 | Special sends | - |

The range 128-175 encodes message sends with 0-2 arguments, which tend to be very common in Pharo code.
The selector's offset is encoded as part of the first byte.

| Bytes | Description | Arguments |
| --- | --- | --- |
| 128-143 | Send 0 argument message, selector at literal `x` | x = byte0 && 0xF |
| 144-159 | Send 1 argument message, selector at literal `x` | x = byte0 && 0xF |
| 160-175 | Send 2 argument message, selector at literal `x` | x = byte0 && 0xF |

The range 176-199 encodes short jump instructions both conditional and unconditional.
The jump offset is encoded as part of the first byte.
Short unconditional jumps can only be forward jumps.
Backjumps (and thus loops) need to be encoded with the longer version (237)

| Bytes | Description | Arguments |
| --- | --- | --- |
| 176-183 | Unconditional jump to offset `x` | x = 0x7 |
| 184-191 | Coditional jump to offset `x` if top = `true` | x = byte0 && 0x7 |
| 192-199 | Coditional jump to offset `x` if top = `false` | x = byte0 && 0x7 |

The range 200-215 encodes _store and pop_ super instructions, which are very common when using single-statement assignments.

| Bytes | Description | Arguments |
| --- | --- | --- |
| 200-207 | Pop and store into receiver's variable `x` | x = byte0 && 0x7 |
| 208-215 | Pop and store into temporary at `x` | x = byte0 && 0x7 |

#### General forms

Instructions that cannot be encoded in the previous ranges -- _e.g.,_ push instance variable number 20 -- can be encoded with a more general form.
General forms can go beyond the limits of byte argument using extensions as described before.
The following tables illustrates such general forms.

| Bytes | Description |
| --- | --- |
| 226 | Push receiver's instance variable at offset `x` |
| 227 | Push literal variable value at offset `x` |
| 228 | Push literal constant at offset `x` |
| 229 | Push temporary variable at offset `x` |
| 232 | Push Integer `x` |
| 233 | Push Character with code point `x` |
|     | x = byte1 + extB << 8 |

| Bytes | Description |
| --- | --- |
| 234 | Send `x` argument message with selector at literal offset `y` |
|     | `x = byte1 && 0x7 + extB << 3, y = byte1 >> 3 + extA << 5` |
| 235 | Super send `x` arguments with selector at literal offset `y`
|     | `x = byte1 && 0x7 + extB << 3, y = byte1 >> 3 + extA << 5` |
|     | Directed if `extB > 64` |
| 237 | Unconditionally jump to offset `x` |
|     | `x = byte1 + (extB << 8)` |
| 238 | Jump to offset `x` if stack top is `true` |
|     | `x = byte1 + (extB << 8)` |
| 239 | Jump to offset `x` if stack top is `false`|
|     | `x = byte1 + (extB << 8)` |

#### Other instructions

Finally some instructions that are rare enough do not have this distinction between the general and the optimized case.
Some of the most notable instructions are:

- **231 - Push array:** boxes the top `x` elements in the stack into an array, and pushes the array to the stack. Depending on the second byte, this bytecode may pop the `x` elements from the stack or not.
- **249 - Push closure:** Creates a closure object that captures the surrounding lexical context.
- **82 - Push pseudo-variable:** Pushes either `thisContext` if `extB = 0`, `thisProcess` if `extB = 1`, undefined otherwise.

### Conclusion

In this chapter we studied how the Pharo virtual machine represents code.
The Pharo VM defines a stack machine: computation is expressed by manipulating a stack with push and pop operations.
Code is organized in methods. Methods contain at most one primitive, a sequence of bytecode instructions and a list of literal values.

We explored many optimizations that can be done at the level of bytecode encoding.
Encoding optimizations help make methods shorter by having smaller bytecode sequences and less method literals.

In the following chapters we will study the implementation of the Pharo interpreter and several of its portable optimizations.
In later chapters we will study low-level optimizations of the interpreter thanks to the Slang framework that applies indirect threading, inlinings, and variable autolocalization.

### References

- Cite The blue book
- Cite the sista paper
- Cite Piumarta's super instructions
- Cite crafting interpreters