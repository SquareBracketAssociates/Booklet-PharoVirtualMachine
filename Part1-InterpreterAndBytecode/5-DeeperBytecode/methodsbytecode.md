## Pharo Bytecode Design

In this chapter we will describe the bytecode in Pharo in detail.

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
For example, we can take the most common bytecode from the Pharo12 release (build 1521) using the script in Listing *@commonbytecode@*.
The script takes all the compiled code (methods and blocks), decodes all their instructions and groups them by their bytes.

```caption=Obtaining the most common bytecode instructions.&anchor=commonbytecode
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
Indeed, amongst the 255 most common instructions, 183 are encoded as a 1 byte instruction.

#### Encoding of Single-byte instructions

Instructions such as `pop` or `push self` are single instructions that do not need any parameter.
The encoding of these instructions is straight forward: they are given a single byte.
For example, `pop` is encoded as 216, while `push self` is encoded as 76.

However, there are other common instructions that have parameters.
This is the case, for example, for the `push instance variable` bytecode that is parameterized with the index of the reference slot in the receiver (the instance variable) to push.
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
A naive translation of `^self` could use the sequence of instructions in Listing *@returningself@*.

```caption=A common bytecode sequence for returning self.&anchor=returningself
push self
return top
```

Using two instructions requires -- at least -- two bytes, and forces the interpreter to pay twice the cost of instruction fetch/decode.
Pharo optimizes such common sequences using a single (often also single-byte) instruction to do the entire operation, often called (static) super instructions in the literature {!citation|ref=Pium98a!}.

#### Optimising for Common Messages and Literals

Another source of overhead happens on the over-reliance on literals.
In Pharo, each method has its own literal frame: literals and constants are not shared between methods, causing a potential redundancy and memory inefficiency.

One way to minimize such overhead is to design special instructions for well-known constants.
Constants such as `nil`, `true`, `false` need to be known by the VM for several tasks such as initializing instance variables, or interpreting conditional jumps.
The VM benefits from this knowledge to provide specialized instructions such as `push true` that do not fetch the `true` object from the method literal frame but from the pool of constants known from the VM.

In the same vein, immediate objects can be crafted by the VM on the fly, avoiding the storage in the literal frame.
Instructions such as `push 0`, encoded as 80, represent the usage of constants that appear often, for example, in loops.
When executing those instructions, the VM creates an immediate object by tagging a well-known value.

Finally, another variation of this optimization happens on common message sends e.g., arithmetic and comparisons selectors.
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
- all variables, temporary and instance, use 0-based indexing. Thus, the bytecode to read the first instance variable is `push instance variable 0`. Similarly, the bytecode to read the first temporary variable is `push temporary variable 0`.


#### Temporary Variables vs Arguments

The Sista bytecode set inherits, mostly for historical reasons, several traits from previous bytecode design.
One particularly interesting trait is that method arguments are modelled as the first (read-only) temporary variables in a method.
For example, while the method in Listing *@tempsversusarguments@* has syntactically one argument and one temporary variable, the underlying implementation will have two temporary variables, from which the first is an argument.

```caption=Arguments are the first temporaries in a method.&anchor=tempsversusarguments
MyClass >> methodWithOneArgAndOneTemp: arg

	| temp |
	...

(MyClass >> #methodWithOneArgAndOneTemp: ) numArgs. "1"
(MyClass >> #methodWithOneArgAndOneTemp: ) numTemps. "2"
```

This decision impacts the bytecode design in different ways.

1. First, to get the real number of temporaries in a method we need to substract the number of arguments from it. See Listing *@numberoftemporaries@*.

2. We need to know the number of arguments of a method to index its temporaries. For example, reading the nth real temporary variable in a method, we need to read the temporary at offset `numArgs + nth - 1` (remember that we need to substract 1 because variable indexes are 0-based).

```caption=Obtaining the real number of temporaries from a method.&anchor=numberoftemporaries
realNumberOfTemporaries := aMethod numTemps - aMethod numArgs
```

#### Bytecode Extension Prefixes

Some bytecode instructions are limited by the encoding: for example, 2-byte instructions usually use one byte as opcode and one byte as argument, limiting the argument to a maximum of 255 values.
For example, the code in Listing *@jumpiffalse@* illustrates the code of the `long jump if false`, that jumps to a given target bytecode if false is found in the stack. This 2-byte bytecode uses the second bytecode as a relative offset from the current bytecode.
Such a restriction can be too limiting for some applications. In the case of our example, this forbids us from having jumps longer than 255 bytes.

```caption=Sketching the jump if false.&anchor=jumpiffalse
extJumpIfFalse
	| byte offset |

	byte := self fetchByte.
	offset := byte.
	self jumplfFalseBy: offset
```

To solve this issue, the sista bytecode includes two prefix instructions: `extension A` and `extension B`.
Prefix instructions prefix normal instructions and work as meta-data for the next instruction: the semantics of a prefix depends on each instruction. Also, instructions that use another instruction _consume it_, zeroing its value for subsequent instructions. See Listing *@jumpiffalsewithextensions@*.
The actual implementation of the `long jump if false` bytecode uses the value of its prefix (if any) to reach further jump offsets.

```caption=Sketching the jump if false with extensions.&anchor=jumpiffalsewithextensions
extJumpIfFalse
	byte := self fetchByte.
	offset := byte + (extB << 8).
	numExtB := extB := extA := 0.
	self jumplfFalseBy: offset
```

The example in Listing *@jumpmorethan255bytes@* shows two bytecode sequences, where the first jumps forward by 255 bytes while the second one jumps forward by 256 bytes.
The first sequence does not require any extensions, while the second one uses an extension.
Notice that the `long jump if false` computes its offset as `byte + (extB << 8)`.
Thus, to compute an offset of 256, we should have an extension of value `1`, and a jump argument of value `0`.

```caption=Jumping more than 255 bytes.&anchor=jumpmorethan255bytes
"Jump forward 255 bytes"
extJumpIfFalse 255

"Jump forward 256 bytes"
extA 1
extJumpIfFalse 0
```

In the case before, the prefix allows jumps to reach jump targets up to `65535` (`255 + (255 << 8)`).
To support larger values, prefixes can be composed: an instruction can have many prefixes that cummulate.
Listing *@extensionimplementation@* shows the definition of the  `extension A` bytecode, which takes the previous value of the extension A, shifts it 8 bits to the left and adds it to the given value.

```caption=The extension implementation.&anchor=extensionimplementation
extABytecode
	extA := (extA bitShift: 8) + self fetchByte.
	self fetchNextBytecode
```

Extensions are composed by adding many prefixes to a given instruction.
For example, the example in Listing *@jumpmorethan65535bytes@* shows a jump with two extensions of 1 and 2, to a jump with argument 3.
This computes the jump offset of 66051 with the formula `(((1 << 8) + 2) << 8 + 3)`.

````caption=Combining extensions to jump above 65535 bytes.&anchor=jumpmorethan65535bytes
"Jump forward 66051 bytes"
extA 1
extA 2
extJumpIfFalse 3
```

#### Super sends

`super` sends deserve a special section for their own, because the Sista Bytecode set introduces _directed_ super sends in addition to the traditional ones.
When using the `super` keyword in Pharo, a message send is issued starting the method lookup from the superclass of the current method's class instead of the receiver's class.
This means that to perform a super send lookup, we need to access the current method's superclass, which should be encoded in the method or bytecode in some way.
Pharo's bytecode set allows for encoding such information in two ways:

- **Per-method encoded super sends:** Traditionally, each Pharo method contains as last literal a reference to its class binding. When performing a normal super send, the algorithm fetches this last literal, then its superclass, and starts the lookup algorithm from there.
- **Per-call-site encoded super sends:** _Directed super sends_ allow to specify as a stack argument the class from where to start the lookup. Directed super sends allow to specify different lookup classes per call-site, by pushing the lookup-class as the last element on the stack. Although initially meant for super sends, this bytecode can be used to control message sends per-call-site at the expense of larger bytecode and literal frames. See Listing *@directedsend@*.

```caption=A directed send bytecode sequence.&anchor=directedsend
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

In this section, we will refer to `byte0` as the first byte of a instruction, `byte1` as the second byte (if present) and `byte2` as the third byte (if present).

#### Sista Optimized Bytecode

The range 0-75 encodes four different families of very common instructions: push receiver instance variable, push literal variable, push literal constant and push temporary variable.
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
| 184-191 | Conditional jump to offset `x` if top = `true` | x = byte0 && 0x7 |
| 192-199 | Conditional jump to offset `x` if top = `false` | x = byte0 && 0x7 |

The range 200-215 encodes _store and pop_ super instructions, which are very common when using single-statement assignments.

| Bytes | Description | Arguments |
| --- | --- | --- |
| 200-207 | Pop and store into receiver's variable `x` | x = byte0 && 0x7 |
| 208-215 | Pop and store into temporary at `x` | x = byte0 && 0x7 |

#### General forms

Instructions that cannot be encoded in the previous ranges -- e.g., push instance variable number 20 -- can be encoded with a more general form.
General forms can go beyond the limits of byte arguments by using extensions as described before.
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

Finally some instructions that are rare enough do not have the distinction between the general and the optimized case.
Some of the most notable instructions are:

- **231 - Push array:** boxes the top `x` elements in the stack into an array, and pushes the array to the stack. Depending on the second byte, this bytecode may pop the `x` elements from the stack or not.
- **249 - Push closure:** Creates a closure object that captures the surrounding lexical context.
- **82 - Push pseudo-variable:** Pushes either `thisContext` if `extB = 0`, `thisProcess` if `extB = 1`, undefined otherwise.

### Conclusion

In this chapter we studied the actual encoding of Pharo instructions.
Moreover, we explored many optimizations that can be done at the level of bytecode encoding.

- Pharo's bytecode set has a variable encoding with instructions taking between 1 and 3 bytes.
- Encoding optimizations make methods shorter by having smaller bytecode sequences and less method literals.
- Common bytecode instructions can be shortened and made special instructions, avoiding expensive literals and arguments.
- Common bytecode sequences can be combined into (shorter!) super instructions too.

In the following chapters we will study the implementation of the Pharo interpreter and several of its portable optimizations.
In later chapters we will study low-level optimizations of the interpreter thanks to the Slang framework that applies indirect threading, inlinings, and variable autolocalization.
