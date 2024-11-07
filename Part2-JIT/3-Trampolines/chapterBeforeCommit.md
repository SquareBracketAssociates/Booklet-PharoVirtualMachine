The input is: [https://rmod-files.lille.inria.fr/Team/Presentations/VMPresentations/VM-Runtime.pdf](https://rmod-files.lille.inria.fr/Team/Presentations/VMPresentations/VM-Runtime.pdf)

[https://rmod-files.lille.inria.fr/Team/Texts/Papers/Kale17a-IWST-VMProfiler.pdf](https://rmod-files.lille.inria.fr/Team/Texts/Papers/Kale17a-IWST-VMProfiler.pdf)
## Runtime Interactions


### The Two Stacks

### Trampolines and enilopmarts
Trampolines facilitate method invocation and transitioning between execution contexts, they are used as jumps from generated native code to c code.

Reverse trampolines are the jumps to (cog methods/ non-compiled Methods / c runtimecode), while running c runtime code, if it is not possible to stay in the native code it will get back to the native code. For example if there is no space left, the garbage collector is solicited via trampoline.

Trampolines are used in slow execution paths that require the vm to prepare for method execution. For example if the method to be called is not cached and needs to be looked up dynamically, the VM will take longer to invoke the method.
Other use cases are :
	- send a message to non-jitted methods, since they don’t have optimised code and have to be executed with the byte-code interpreter, The interpreter tries to keep this execution mode for as long as possible.
	- PIC, polymorphic inline caching .
	- Calling the garbage collector, for example if new message doesn’t find space left, the GC is called via a trampoline to the c runtime code, because it is not a compiled.
	- Slow allocations.
	- Immutability checks, which are guards to check that targeted objects by the message is not being modified elsewhere .
    _ Discovery routine at vm start-up. to know which instructions the processor supports.


Reverse trampolines are used to call jitted methods, and low level routines like getting the state of the machine.

### Calling Convention

check marshallSendArguments:

The c functions are compiled by c compiler, and we don’t know which registers they use, We have to be careful using them since they could do anything. The generated native code runs in a separate stack and doesn’t follow the C calling convention. 

the current production code generator  is the StackToRegisterMappingCogit, it takes advantage of registers fast accessibility for operations and only pushes operands to the stack when register resources are all busy. For exampple it implements a register-based calling convention for low-arity sends

- if the number of arguments is less or equal to numRegArgs then the receiver and arguments are passed in registers
"bring the code of the sends with registers and how many arguments can be passed for registers only calling convention i think it is under 8, if more the other arguments are pushed to the stack"

	


#### Calling Interpreter from Interpreter

Receiver and parameters are pushed to the stack, then a lookup of the method is done.

#### Calling MC from MC

In c the the calling function, which usually starts with main, transfers control of the stack to the called functions by expanding the stack with a frame at the next available high address, the stack shrinks after the called function pops its variables from the stack, giving the result to the caller which continues its execution at the return address.

#### Calling MC from Interpreter

There are two ways to jump, by return, by call.


VM C runtime calls a reverse trampoline (enilopmart) that prepares arguments and switches to smalltalk stack and returns to the generated method not by calling it to avoid having reverse trampoline in the stack.

(if a machine code method contains a call to an interpreter primitive it will push any
		register arguments (and if on a RISC, the return pc from the LinReg) on the stack before calling
		the primitive so that to the primitive the stack looks the same as it does in the interpreter. ) Copied from StackToRegisterMappingCogit documentation
Reverse trampolines take arguments through the smalltalk stack, pop them and puts them in the right spots (e.g., registers) then returns to the generated method.


#### Calling Interpreter from MC

Reentering the interpreter, sigjmp, setjump.

To make the transition the native code calls a trampoline that prepares arguments and aligns the stack, sets up conditions to then call the c code if those are satisfied.
### Exiting MC Slow paths

must be boolean, store checks, write barriers...


