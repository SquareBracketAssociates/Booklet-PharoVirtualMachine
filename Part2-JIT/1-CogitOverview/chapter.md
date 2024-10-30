## The Cogit JIT Compiler

Commonly executed methods in Pharo are compiled to machine code by the Cogit just-in-time (JIT) compiler.
The Cogit JIT compiler is bytecode-to-machine code compiler in the VM, whose main objective is to remove interpretation overhead.
It implements peephole optimizations through abstract interpretation, and uses an x86-inspired intermediate representation with fixed virtual registers.
In this chapter we give a general view on the compiler architecture, its intermediate representation, the different compilation passes.

### The Cogit Architecture

The Cogit JIT compiler is a method compiler.
This means that the unit of compilation is an entire method, as opposed to other compilers that compile traces or regions of code.
This also means that methods are compiled on their entirety or they are not compiled at all.

#### Compilation Phases

The compilation process is organized in three main phases:
1. **Semantic Analysis.** The method bytecode's is semantically analysed to extract static properties. For example, it determines if the method needs a frame or not.
2. **Intermediate Code Generation.** The method bytecode is iterated one by one, creating intermediate code for each bytecode instruction.
3. **Machine Code Generation.** The intermediate code is iterated to generate corresponding machine code.

#### Table Dispatch (again)

The first two phases of compilation use the same table dispatch mechanism as in the interpreter.
That is, a table contains, for each valid opcode, a function to execute.
In the case of the compiler, the function to execute is a _code generator_, a function that generates the intermediate code for each bytecode.
Moreover, each entry in the table is accompanied by annotations that guide compilation.

The following code snipped illustrates the compiler's dispatch table.
Each entry in the table has four mandatory fields, followed by zero or more annotations.
The first mandatory field is the number of bytes of the instruction.
The second and third mandatory fields are the range of instructions for that generator: many bytecode can share the same generator, if they have the same behavior.
The fourth mandatory field is the _generator_ function.

```
(
  (1   0   15 genPushReceiverVariableBytecode isInstVarRef)
  (1  16   31 genPushLiteralVariable16CasesBytecode	needsFrameNever: 1)
  ...
)
```

In this example we see that the first 16 instructions use the generator `genPushReceiverVariableBytecode`, which is annotated to say it references instance variables.
In the second entry, instructions 16 through 31 use the generator `genPushLiteralVariable16CasesBytecode`, do not need a frame, and is explicitly annotated that it pushes one element to the stack.

### Semantic Analysis

The first stage of compilation is iterate over all the bytecode instructions of a method to extract metadata that will serve for the rest of the compilation process. Bytecode is _scanned_ from first to last

#### Unknown/Uncompilable bytecode instructions

Cogit is an all-or-nothing method compiler.
Its unit of compilation is the method: all of its instructions are considered for compilation from begin to end.
Thus, to compile a method, all of its bytecode instructions should have a corresponding translation.

In the current state, all bytecode instructions are defined in the compiler.
However, to support an iterative development and experimental features, Cogit allows that some bytecode do not contain a generator function.
In that case, the instruction cannot be compiled: compilation aborts and execution continues in the interpreter.

**Analysis Implementation:** As soon as one uncompilable/unknown bytecode is found, compilation is aborted.

#### The last valid bytecode instruction

As explained in the chapter on methods, methods contain an arbitrary long trailer at the end of their variable part, and no mark of where instructions finish and the trailer starts. This is not an issue in the interpreter, where instructions are evaluated one by one until a `return` instruction is found. To avoid compiling the trailer bytes as instructions, the compiler analyses the code to find what is the last reachable instruction in the method.

**Analysis Implementation:** The analysis starts from the first program counter and finishes on the last return. Track the _farthest jump_, _i.e.,_ the maximum target program counter of all jump bytecodes found. When a return instruction is found, if the return instruction is after the farthest jump, this is the last valid bytecode.

#### Frameless methods
Not all methods require a frame. Methods require a frame if they perform message sends, use temporary variables or their context is captured by `thisContext` or block closures. Knowing what methods require a frame allows an extremely useful optimization: the compiled code can avoid creating an expensive frame if not necessary.

**Analysis Implementation:** If any of the method's bytecode is annotated to need a frame, mark the method as frameful.

#### Backward branch target instructions

The semantic analysis identifies in a first pass all bytecode instructions that are targetted by a backwards jump.
Backward jumps jump to instructions _before_ them, and are found after their targets.
Identifying their targets before IR generation help the compiler manage control flow merge points and minimize the mappings stored between original instructions and generated IR instructions.

**Analysis Implementation:** For each a branch has a negative target, store it's target pc in a table for later treatment.


### Intermediate Representation: the Cogit Register Transfer Language

After extracting a method's meta-data, Cogit translates bytecode instructions into its own intermediate representation, named Cogit Register Transfer Language or CogitRTL for short. This sections explains the key design points of the intermediate language and later its implementation.

#### An Internal Domain Specific Language

CogitRTL is designed as an internal Domain Specific Language (DSL) to create a list of instructions.
The list starts empty and new instructions are appended to the end of the list.
The DSL expresses instructions as messages to the `cogit` instance.
Such messages work as _factory methods_, creating IR instructions and appending them to the instruction list.
Instructions have operands to operate upon: registers, constant values and memory addresses.
For example, the following example shows how to generate code that tags an integer to create a 64-bit tagged small integer.

```
"Assume integer is on TempReg"

"First shift it to the left by 3 bits"
cogit LogicalShiftLeftCq: 3 R: TempReg.

"Add the small integer tag to it (1)"
cogit AddCq: 1 R: TempReg.
```

CogitRTL defines, at the time of this writing, 164 different instructions.
Instructions are organized in several groups, with the most prominent being:
 - **Arithmetic instructions** implement addition, subtraction, etc.
 - **Bitwise instructions** implement bit and, bit or, shifts.
 - **Call instructions** implement instruction to invoke procedures, typically named `call` or `branch with link` in RISC architectures.
 - **Comparison instructions** implement the comparison of magnitures.
 - **Jump instructions** implement control flow instructions, both conditional and unconditional.
 - **Load/Store instructions** implement instructions that move values from memory to registers and vice-versa,
 - **Copy instructions** implement instructions that move values between registers.
 - **Stack instructions** implement instructions that push/pop values to the stack.

#### Two Address Code: Intel's Legacy

#### Fixed Virtual Registers

#### Naming Convention and Addressing Modes

CogitRTL uses a very strong naming convention to identify IR factory methods and the type of operands.
First, instruction factory methods are named in uppercase.
Moreover, the keywords used for the arguments denote the kind of argument in question.
Instruction operands have a type and a bit length.
The operand type determines where data is fetched/stored from/to: _e.g.,_ a register, memory, or the instruction itself.
The bit-length, used for constant and memory operands, specifies how many bytes will be fetched/stored: _e.g.,_ a byte, a machine word, a fixed 32-bit value.


Registers operands are denoted by keywords starting with **R** and use a full-length register, thus bit-length need not to be specified.
For example, the following instruction adds the values of `Register1` and `Register2`, putting the result in `Register2`.
```
cogit AddR: Register1 R: Register2.
```

On the other hand, constant and memory operands need to specify the required bit-length.
Constant operands are denoted by keywords starting with **C**.
Memory operands are of three kinds: 

- **Absolute addresses.** Denoted by the **A** prefix. They indicate that the operand is an absolute address to operate with.
For example, the following code loads the value found at address 100 to `Register1`.

```
cogit MoveA: 100 R: Register1.
```

- **Relative addresses.** Denoted by the **M** prefix. They indicate that the operand is a memory address found relative at a fixed offset from a base.
Relative address operands have two arguments for the base and offset, indicated in lowercase.
For example, the following code loads a word from the memory address `BaseRegister`+`offset` into `DestinationRegister`.

```
cogit MoveMw: 100 r: BaseRegister R: DestinationRegister.
```

- **Scaled addresses,** Denoted by the **X** prefix. They indicate that the operand is a memory address found relative at an offset from a base, with this offset called an index. Both base and index are represented by registers. The difference between relative and scaled addresses is that scaled addesses multiply (scale) the index by the size of data. This means that loading/storing a word multiplies the offset by 8 in 64-bit machines. For example, the following code stores the word in `Register1` into the memory address `BaseReg`+`IndexReg` * 8.

```
cogit MoveR: Register1 Xwr: IndexReg R: BaseReg.
```

We have already seen in the examples above that specifying the bit-length is mandatory in all operands except for registers operands.
Bit-length forces that values are encoding in _at least_ the specified size.
Such sizes can be `b` for byte, `16`, `32`, `64` for 16/32/64 bits respectively, and `w` word size.
For example, arguments with keyword **Aw** are memory addresses that load/store a machine word.

Finally, constant instructions have an additional
We will see in the chapter on inline caches that controlling how large is an instruction, how values are encoded and where they are in machine code is useful for code patching techniques.




The following examples show how we can increment a register by 1, encoding the 1 in different sizes, and the corresponding arm64 assembly code.
The first example, using a quick encoding, uses a move instruction, encoding the immediate value `1` in the instruction itself.
The second example, using a word encoding: it uses a load instruction that reads a memory address (8 bytes relative to the current program counter), and the value is stored in memory at that address by the compiler.

```
"Generate the following arm64 assembly: 
  mov	x1, #1"
cogit MoveCq: 1 R: Register1.

"Generate the following arm64 assembly: 
  ldr	x1, #8"
cogit MoveCw: 1 R: Register1.
```



- Arguments denoting instruction operands start in uppercase.
- Arguments starting in lowercase are part of the previous operand

```
cogit AddR: 1 R: TempReg.
```


### Intermediate Code Generation

#### Bytecode

#### Primitives

### Machine Code Generation

#### PC Mapped Instructions

Method execution can be suspended on so called _suspension points_.
A suspension point is a point in the execution where the observable state of the program is coherent.
In contrast, language implementations can apply any optimization and transformation in between suspension points, for example, breaking potential invariants for performance.

In Pharo, suspension points are message sends and backjumps.
When a method is suspended on machine code, a debugger can be opened on the method, or a context-switch may happen, making the current thread and stack available to the rest of the program, which expects bytecode program counters.
Thus, the bytecode program counter must be made available at such points.

For this purpose, the compiler stores meta-data in its trailer, containing a mapping `machinecode program counter -> bytecode program counter`