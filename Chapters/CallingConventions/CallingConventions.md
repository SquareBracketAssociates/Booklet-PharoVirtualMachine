# Calling Conventions

A calling convention dictates how two procedures communicate.
This has two main points:

 - first, how arguments are passed between caller and callee (by reference, by copy...), how the procedure returns
 - second, how limited resources are maintained

The principle is that procedures are black boxes.
A procedure does not know the shape of its caller, nor the shape of its callee.
The caller may be optimized differently, use a different/unconventional set of registers.
This means that a procedure must be written to be called from anywhere and to call procedures that can do anything.


## Passing Arguments

The convention dictates how arguments are passed, and where they are stored.
This way, the convention decouples procedures from their implementations.
For example, the Smalltalk-80 calling convention dictates that upon a message send, the receiver and all arguments are pushed to the stack. Then the method executed, which knows it has N arguments by construction, can access the receiver (self) by skipping the N top elements in the stack.

## Returning

Low-level architectures store the current program counter in a special CPU register.
The program counter register is unique, and can only hold a single instruction pointer, which for efficiency reasons is made the program counter of the currently executing procedure.
This means that the program counters of all the procedures active on the call stack must be stored somewhere, and restored when control returns to those procedures.

A calling convention dictates how the current program counter is stored when a procedure is called, how the control is passed to the called procedure, and how the program counter is restored when the procedure returns. There are two main families of solutions for this aspect in low-leve ISAs (Instruction set architectures).

- In CISC (complex instruction set architectures) machines, the `call procedure` instruction will push the current program counter to the stack and transfer the control to the procedure. The return instruction will do the inverse. In pseudocode:


```
call procedure
=>
push IP
IP := procedure

return
=>
IP := pop
```

- In RISC (reduced instruction set architectures) machines, the `call procedure` instruction will copy the current program counter to the link register and transfer the control to the procedure. The return instruction will do the inverse. This register must be saved by the callee explicitly if needed.


```
call procedure
=>
LR := IP
IP := procedure

return
=>
IP := LR
```



## Shared State

When procedure A calls procedure B, A does not know what potential effects B will produce.
In general, the problem lies in the usage of registers and the call stack.
If procedure A was using registers R0 and R1, it cannot know if procedure B will read from those registers or write on them.
Thus, procedure A should make sure that its state before the call is preserved after the call returns.

### Keeping Registers

Calling conventions dictate how such preserving must be done.
In general, registers are split into two sets: caller saved registers, and callee saved registers.
A caller-saved register is a register that the caller must preserve before the call and restore after the call.
A callee saved register is a register that the callee must preserve when it's called and restore before it returns.
Preserving and restoring a register is usually done by saving the values in predictable memory positions, usually on a stack.

### Keeping the Call Stack

The same problem arises with the call stack. A procedure calling another procedure must not only assume that the registers it was using were not modified, but also it must assume that the stack was preserved.
This is particularly important when calling high-order functions, closures, or polymorphic procedures.
Otherwise, if each procedure leaves the registers and the stack in different states, the caller will not be able to continue correctly.

## How Calling Conventions Change per Platform


## Calling Convention in the VM

The interpreter and the JIT share the same calling conventions.



![....](figures/20220928_114230.jpg width=100&label=send)

![....](figures/frames.jpg width=100&label=frames)

