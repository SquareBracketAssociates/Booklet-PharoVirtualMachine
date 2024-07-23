## The Interpreter


The virtual machine contains one interpreter and a compiler \(that compiles to assembly code\). Here we talk about the interpreter. It executes the bytecode instructions found in CompiledMethods. The interpreter uses five pieces of information refered hereafter as the state of the interpreter and repeatedly performs a three-step cycle.

### Interpreter Overview

#### Interpreter State

- The CompiledMethod whose bytecodes are being executed.
- The location of the next bytecode to be executed in that CompiledMethod. This is the interpreter's _instruction pointer_.
- The receiver and arguments of the message that invoked the CompiledMethod.
- Any temporary variables needed by the CompiledMethod.
- A stack.


The execution of most bytecodes involves the interpreter's stack. Push bytecodes tell where to find objects to add to the stack. Store bytecodes tell where to put objects found on the stack. Send bytecodes remove the receiver and arguments of messages from the stack. When the result of a message is computed, it is pushed onto the stack.

#### Instruction Dispatch

- Fetch the bytecode from the CompiledMethod indicated by the instruction pointer.
- Increment the instruction pointer.
- Perform the function specified by the bytecode.


As an example of the interpreter's function, we will trace its execution of the CompiledMethod  `Rectangle>>#width`.
The state of the interpreter will be displayed after each of its cycles.
The instruction pointer will be indicated by an arrow pointing at the next bytecode in the CompiledMethod to be executed. 
 
- **==>** <01> pushRcvr: 1 - push the value of the receiver's second instance variable \(corner\) onto the stack



The receiver, arguments, temporary variables, and objects on the stack will be shown as normally printed. 
For example, if a message is sent to a Rectangle, the receiver will be shown as:

- **Receiver:**	50@50 corner: 200@200 



At the start of execution, the stack is empty and the instruction pointer indicates the first bytecode in the CompiledMethod. This CompiledMethod does not require temporaries and the invoking message did not have arguments, so these two categories are also empty. 

#### Step 1.

The interpreter is in an initial state. It points to the next instructions to be executed. It knows the receiver of the method to be executed.

Rectangle >> #width
- **==>** <01> pushRcvr: 1 - push the value of the receiver's second instance variable \(corner\) onto the stack
- <7E> send: x - send the unaay message  x
- <00> pushRcvr: 0 - push the value of the receiver's first instance variable \(origin\) onto the stack
- <7E> send: x  - send the unary message x
- <61> send: -  - send the binary message -
- <5C> returnTop - return the object on top of the stack as the value of the message \(width\) 
- **Receiver:** 50@50  corner: 200@200
- **Arguments:**
- **Temporary Variables:**
- **Stack:**


#### Step 2.

Following one cycle of the interpreter,  the value of the receiver's second instance variable has been copied onto the stack and the instruction pointer has been advanced.

Rectangle >> #width
- <01> pushRcvr: 1 - push the value of the receiver's second instance variable \(corner\) onto the stack
- **==>** <7E> send: x - send the unray message with the selector x
- <00> pushRcvr: 0 - push the value of the receiver's first instance variable \(origin\) onto the stack
- <7E> send: x  - send the unary message with the selector x
- <61> send: -  - send the binary message with the selector -
- <5C> returnTop - return the object on top of the stack as the value of the message \(width\) 
- **Receiver:** 50@50 corner: 200@200
- **Arguments:**
- **Temporary Variables:**
- **Stack:** 200@200



#### Step 3.

The interpreter encounters a send bytecode. 
It removes one object from the stack and uses it as the receiver of a message with selector `x`. 
Sending a message will not be described in detail here. 
Sending messages will be described in later sections.
For the moment, it is only necessary to know that eventually the result of the `x` message will be pushed onto the stack. 

Rectangle >> #width
- <01> pushRcvr: 1 - push the value of the receiver's second instance variable \(corner\) onto the stack
- <7E> send: x - send the unray message with the selector x
- **==>** <00> pushRcvr: 0 - push the value of the receiver's first instance variable \(origin\) onto the stack
- <7E> send: x  - send the unary message with the selector x
- <61> send: -  - send the binary message with the selector -
- <5C> returnTop - return the object on top of the stack as the value of the message \(width\) 
- **Receiver:** 50@50 corner: 200@200
- **Arguments:**
- **Temporary Variables:**
- **Stack:** 200


#### Step 4.

In this cycle, the value of the first receiver's first instance variables \(origin\) onto the stack

Rectangle >> #width
- <01> pushRcvr: 1 - push the value of the receiver's second instance variable \(corner\) onto the stack
- <7E> send: x - send the unray message with the selector x
- <00> pushRcvr: 0 - push the value of the receiver's first instance variable \(origin\) onto the stack
- **==>** <7E> send: x  - send the unary message with the selector x
- <61> send: -  - send the binary message with the selector -
- <5C> returnTop - return the object on top of the stack as the value of the message \(width\) 
- **Receiver:** 50@50  corner: 200@200
- **Arguments:**
- **Temporary Variables:**
- **Stack:** 200
- **Stack:** 50@50 



#### Step 5.

In this cycle, the message x is sent: it removes one object from the stack and uses it as the receiver of a message with selector `x` and push back the result on the stack.

Rectangle >> #width
- <01> pushRcvr: 1 - push the value of the receiver's second instance variable \(corner\) onto the stack
- <7E> send: x - send the unray message with the selector x
- <00> pushRcvr: 0 - push the value of the receiver's first instance variable \(origin\) onto the stack
- <7E> send: x  - send the unary message with the selector x
- **==>** <61> send: -  - send the binary message with the selector -
- <5C> returnTop - return the object on top of the stack as the value of the message \(width\) 
- **Receiver:** 50@50  corner: 200@200
- **Arguments:**
- **Temporary Variables:**
- **Stack:** 200
- **Stack:** 50


#### Step 6.

The final bytecode returns a result to the width message. The result is found on the stack \(150\). It is clear by this point that a return bytecode must involve pushing the result onto another stack. 
The details of returning a value to a message will be described after the description of sending a message.


Rectangle >> #width
- <01> pushRcvr: 1 - push the value of the receiver's second instance variable \(corner\) onto the stack
- <7E> send: x - send the unray message with the selector x
- <00> pushRcvr: 0 - push the value of the receiver's first instance variable \(origin\) onto the stack
- <7E> send: x  - send the unary message with the selector x
- <61> send: -  - send the binary message with the selector -
- **==>** <5C> returnTop - return the object on top of the stack as the value of the message \(width\) 
- **Receiver:** 50@50  corner: 200@200
- **Arguments:**
- **Temporary Variables:**
- **Stack:** 150

### Supporting Message sends

#### The Call Stack

#### Calling Convention

The interpreter and the JIT share the same calling conventions. The receiver and arguments are pushed on the stack. The instruction pointer is also pushed to the stack after the arguments. 

Figure *@stackGrowing@* shows that the stack is growing down from high addresses to low addresses.
This convention is important and has an impact of many aspect such as object allocation and different logic in garbage collector implementation.

![Stack growing down.](figures/StackGrowingDown.pdf width=30&label=stackGrowing)


Figure *@beforesend@* shows that the before doing a call, the receiver and arguments are pushed to the stack.

![Receiver and arguments are pushed to the stack.](figures/BeforeSend.pdf width=80&label=beforesend)

Figure *@aftersend@* shows that the instruction pointer (IP) is also pushed to the stack. This way it is possible to find which instruction is the next one to execute on return. Notice also that the interpreter and the VM are __caller-saved__. It means that this is the caller responsibility to store information that should be recovered on return of the called function.

![Caller saved: the IP is also pushed to make sure that the caller can know the next instruction on return.](figures/AfterSend.pdf width=80&label=aftersend)

Figure *@generalArguments@* shows that the framepointer is used to compute 
- method argument. Since the arguments are pushed on the stack before the new frame is allocated, a method argument is always computed as an addition to the framepointer (`arg1 = FP + arg1offset`).
- method. The method (with its metadata) is located at a fix offset from the frame pointer. Hence `method= FP- method offset`.


![arg1 = FP + offset and method = FP - method offset.](figures/GeneralArgument.pdf width=100&label=generalArguments)



#### Method lookup

### Interpreter optimizations

#### Where does the interpreter time go?

#### Global Lookup Cache

#### Static Type Predictions

#### Instruction Lookakeads

#### The Cycle of the Interpreter

