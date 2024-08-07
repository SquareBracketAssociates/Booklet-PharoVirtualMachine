## Basics on Execution: The interpreter point of view


In this chapter we will explain how a compiled method is executed by the virtual machine interpreter. 
In a future chapter we will explain how the code is compiled by the virtual machine compiler. 


We will start with the source methods written by programmers. A method is compiled into sequences of instructions called _bytecodes_.  The bytecodes produced by the compiler are instructions for an interpreter, which is described in the second section. We will present the logic of interpreting the method bytecodes using a dedicated and internal stack for bytecode. 
Then we present contexts that support the execution of messages. 


### Reminder

Imagine the method `Rectangle>>center` defined as follows

```
Rectangle >> width
	"Answer the width of the receiver."
	^ corner x - origin x
```


A programmer does not interact directly with the compiler. When a new source method is added to a class (`Rectangle` in this example), the system asks the compiler for an instance of `CompiledMethod` containing the bytecode translation of the source method. 
The class provides the compiler with some necessary information not given in the source method, including the names of the receiver's instance variables and the dictionaries containing accessible shared variables (global, class, and pool variables). 
The compiler translates the source text into a `CompiledMethod` and the class stores the method in its message dictionary. 

### Bytecodes by example

Source methods are translated by the compiler into sequences of instructions for a stack-oriented interpreter. 
The instructions are numbers called bytecodes. For example, the bytecodes corresponding to the source method shown above are: 1, 126, 0, 126, 97, and 92.

```
(Rectangle >> #width) bytecode
>>> #[1 126 0 126 97 92]
```



Pharo supports a simple description of the bytecode using the message `symbolicBytecodes`.

```
(Rectangle >> #width) symbolicBytecodes  
	25 <01> pushRcvr: 1 
	26 <7E> send: x 
	27 <00> pushRcvr: 0 
	28 <7E> send: x 
	29 <61> send: - 
	30 <5C> returnTop
```



A bytecode's value gives us little indication of its meaning to the interpreter. 
In this chapter, we follow the convention of the Blue Book and accompany lists of bytecodes with comments about their functions. 


We wrap in parentheses any part of a bytecode's comment that depends on the context of the method in which it appears. 
The unparenthesized part of the comment describes its general function. 
For example, the bytecode 0 always instructs the interpreter to push the value of the receiver's first instance variable onto its stack.
The fact that the variable is named `origin` depends on the fact that this method is used by the class `Rectangle`, so `origin` is parenthesized. 
The commented form of the bytecodes for Rectangle width is shown below: 


![.](figures/width_0.pdf)

%Rectangle >> #width
%- <01> pushRcvr: 1 - push the value of the receiver's second instance variable (corner) onto the stack
%- <7E> send: x - send the unary message x
%- <00> pushRcvr: 0 - push the value of the receiver's first instance variable (origin) onto the stack
%- <7E> send: x  - send the unary message x
%- <61> send: -  - send the binary message -
%- <5C> returnTop - return the object on top of the stack as the value of the message (width) 



#### About the stack.

The stack mentioned in some of the bytecodes is used for several purposes:
- In method `width`, it is used to hold the receiver, arguments, and results of the two messages that are sent. 
- The stack is also used as the source of the result returned from the `width` method. 




### A store bytecode


Another example of the bytecodes compiled from a source method illustrates the use of a store bytecode.
Let us define the method `extent:` as follows:

```
Rectangle >> extent: aPoint
	corner := origin + aPoint
```


The message `extent:` changes the receiver's `width` and `height` to be equal to the `x` and `y` coordinates of the argument \(`a Point`\). 
The receiver's upper left corner \(`origin`\) is kept the same and the lower right corner \(`corner`\) is moved.

```
(Rectangle >> #extent:) bytecode 
>>> #[0 64 96 201 88]
```



Rectangle >> #extent: 
- <00> pushRcvr: 0 - push the value of the receiver's first instance variable \(origin\) onto the stack 
- <40> pushTemp: 0 - push the argument \(aPoint\) onto the stack 
- <60> send: `+` - send a binary message + 
- <C9> popIntoRcvr: 1 - pop the top object off of the stack and store it in the receiver's second instance variable \(`corner`\) 
- 29 <58> returnSelf - return the receiver as the value of the message \(`extent:`\) 






### Compiled Methods


The compiler creates an instance of `CompiledMethod` to hold the bytecode translation of a source method. In addition to the bytecodes themselves, a `CompiledMethod` contains a set of objects called its _literal frame_.
The literal frame contains any objects that could not be referred to directly by bytecodes. 

All of the objects in the methods `Rectangle>>#extent:` and `Rectangle>>#width` were referred to directly by bytecodes, so the `CompiledMethod` of these methods do not need literal frames. 

As an example of a `CompiledMethod` with a literal frame, consider the method `Rectangle>>#squishedWithin:`. 
The `squishedWithin:` message changes the receiver fits within `aRectangle` by reducing its size, not by changing its origin.

```
Rectangle >> squishedWithin: aRectangle
	"Return an adjustment of the receiver that fits within aRectangle by reducing its size, not by changing its origin."

	^ origin corner: (corner min: aRectangle bottomRight)
```



```
(Rectangle >> #squishedWithin:) bytecode 
>>> #[0 1 64 128 145 146 92]
```



The message selectors \(`bottomRight`, `min:`, `corner:`\)  are not in the set that can be directly referenced by bytecodes.
These selectors are included in the `CompiledMethod`'s literal frame and the send bytecodes refer to the selectors by their position in the literal frame. 
We show the compiledMethod's literal frame after its bytecodes.
 
`Rectangle>>#squishedWithin:`
- <00> pushRcvr: 0 - push the value of the receiver's first instance variable \(`origin`\) onto the stack 
- <01> pushRcvr: 1 - push the value of the receiver's second instance variable \(`corner`\) onto the stack 
- <40> pushTemp: 0 - push the argument \(`aRectangle`\) 
- <80> send: bottomRight - send a unary message with the selector in the literal frame location \(`bottomRight`\) 
- <91> send: min: - send a binary message with the selector in the literal frame location \(`min:`\) 
- <92> send: corner: - send a keyword message with the selector in the literal frame location \(`corner:`\) 
- <5C> returnTop - return the object on top of the stack as the value of the message

Literal frame
- 1 `#bottomRight` 
- 2 `#min:`
- 3 `#corner:`

	
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



 Any objects referred to in a CompiledMethod's bytecodes that do not fall into one of the categories above must appear in its literal frame. The objects ordinarily contained in a literal frame are
- shared variables \(global, class, and pool\)
- most literal constants \(numbers, characters, strings, arrays, and symbols\)
- most message selectors \(those that are not special\)

Objects of these three types may be intermixed in the literal frame. 


% % we could add the following as in the blue book
% Two types of object that were referred to above, temporary variables and shared variables, have not been used in the example methods. The following example method for Rectangle merge: uses both types. The merge: message is used to find a Rectangle that includes the areas in both the receiver and the argument.
% merge: aRectangle
% | minPoint maxPoint |
% minPoint := origin min: aRectangle origin.
% maxPoint := corner max: aRectangle corner.
% ^Rectangle origin: minPoint corner: maxPoint
% When a CompiledMethod uses temporary variables (maxPoint and minPoint in this example), the number required is specified in %the first line of its printed form. When a CompiledMethod uses a shared variable (Rectangle in this example) an instance of Association is included in its literal frame. All CompiledMethods that refer to a particular shared variable's name include the same Association in their literal frames. 
 
% Rectangle merge: requires 2 temporary variables
% 0	push the value of the receiver's first instance variable (origin) onto the stack
% 16	push the contents of the first temporary frame location (the argument aRectangle) onto the stack 
% 209	send a unary message with the selector in the second literal frame location (origin) 
% 224	send the single argument message with the selector in the first literal frame location (min:) 
% 105	pop the top object off of the stack and stare in the second temporary frame location (minPoint) 
% 1	push the value of the receiver's second instance variable (corner) onto the stack 
% 16	push the contents of the first temporary frame location (the argument aRectangle) onto the stack 
% 211	send a unary message with the selector in the fourth literal frame location (corner) 
% 226	send a single argument message with the selector in the third literal frame location (max:)
% 106	pop the top object off of the stack and store it in the third temporary frame location (maxPoint)
% 69	push the value of the shared variable in the sixth literal frame location (Rectangle) onto the stack 
% 17	push the contents of the second temporary frame location (minPoint) onto the stack 
% 18	push the contents of the third temporary frame location (maxPoint) onto the stack 
% 244	send the two argument message with the selector in the fifth literal frame location (origin:corner:) 
% 124	return the object on top of the stack as the value of the message (merge:) 
% literal frame 
% #min:  
% #origin  
% #max:  
% #corner  
% #origin:corner:  
% Association: #Rectangle -> Rectangle  

### Temporary Variables

Temporary variables are created for a particular execution of a CompiledMethod and cease to exist when the execution is complete. 
The CompiledMethod indicates to the interpreter how many temporary variables are required. 
The arguments of the message and the values of the temporary variables are stored together in the temporary frame.
The arguments are stored first and the temporary variable values immediately after.
They are accessed by the same type of bytecode \(whose comments refer to a temporary frame location\). 
 
The compiler enforces the fact that the values of the argument names cannot be changed by never issuing a store bytecode referring to the part of the temporary frame inhabited by the arguments.

#### Shared Variables.

Shared variables are held in dictionaries or environments.

- global variables in the system dictionary unique instance.
- class variables in a dictionary held in the class declaring them.
- pool variables in collection held in shared pool classes.


The system represents associations in general, and shared variables in particular, with instances of `Association` \(representing key value pair of information\). 
When the compiler encounters the name of a shared variable in a source method, the `Association` with the same name is included in the CompiledMethod's literal frame.
The bytecodes that access shared variables indicate the location of an `Association` in the literal frame. 
The actual value of the variable is stored in an instance variable of the `Association`. 

Note that the literal frame or dictionaries are holding an `Association` and not just the value because this way it does not force a recompilation of all the users of a shared variable when its value change.



### The bytecodes


The interpreter understands  bytecode instructions that fall into five categories: pushes, stores, sends, returns, and jumps. This section gives a general description of each type of bytecode without going into detail about which bytecode represents which instruction. 

Some of the bytecodes take extensions. An extension is one or two bytes following the bytecode that further specify the instruction. An extension is not an instruction on its own, it is only a part of an instruction.

#### Push Bytecodes.  

 A push bytecode indicates the source of an object to be added to the top of the interpreter's stack. The sources include

- the receiver of the message that invoked the CompiledMethod
- the instance variables of the receiver
- the temporary frame \(the arguments of the message and the temporary variables\)
- the literal frame of the CompiledMethod
- the top of the stack \(i.e., this bytecode duplicates the top of stack\)


Examples of most of the types of push bytecode have been included in the examples.
The bytecode that duplicates the top of the stack is used to implement cascaded messages. 

Two different types of push bytecode use the literal frame as their source.
One is used to push literal constants and the other to push the value of shared variables.
Literal constants are stored directly in the literal frame, but the values of shared variables are stored in an Association that is pointed to by the literal frame. 


% The following example method uses one shared variable and one literal constant.
% incrementIndex
% ^Index := Index + 4

% ExampleClass incrementIndex
% 64	push the value of the shared variable in the first literal frame location (Index) onto the stack
% 33	push the constant in the second literal frame location (4) onto the stack
% 176	send a binary message with the selector +
% 129,192	store the object on top of the stack in the shared variable in the first literal frame location (Index)
% 124	return the object on top of the stack as the value of the message (incrementIndex)
% literal frame 
% Association: #Index -> 260 

#### Store Bytecodes.


The bytecodes compiled from an assignment expression end with a store bytecode. The bytecodes before the store bytecode compute the new value of a variable and leave it on top of the stack. A store bytecode indicates the variable whose value should be changed. The variables that can be changed are
- the instance variables of the receiver
- temporary variables
- shared variables


Some of the store bytecodes remove the object to be stored from the stack, and others leave the object on top of the stack, after storing it.


#### Send Bytecodes.

A send bytecode specifies the selector of a message to be sent and how many arguments it should have. The receiver and arguments of the message are taken off the interpreter's stack, the receiver from below the arguments. By the time the bytecode following the send is executed, the message's result will have replaced its receiver and arguments on the top of the stack. The details of sending messages and returning results is the subject of the next sections of this chapter. A set of 32 send bytecodes refer directly to the special selectors listed earlier. The other send bytecodes refer to their selectors in the literal frame.

#### Return Bytecodes.

When a return bytecode is encountered, the CompiledMethod in which it was found has been completely executed. Therefore a value is returned for the message that invoked that CompiledMethod. The value is usually found on top of the stack. Four special return bytecodes return the message receiver \(self\), true, false, and nil.

#### Jump Bytecodes.


Ordinarily, the interpreter executes the bytecodes sequentially in the order they appear in a CompiledMethod. The jump bytecodes indicate that the next bytecode to execute is not the one following the jump. There are two varieties of jump, unconditional and conditional. The unconditional jumps transfer control whenever they are encountered. The conditional jumps will only transfer control if the top of the stack is a specified value. Some of the conditional jumps transfer if the top object on the stack is true and others if it is false. The jump bytecodes are used to implement efficient control structures. 


The control structures that are so optimized by the compiler are conditional selection messages to Booleans \(`ifTrue:`, `ifFalse:`, and `ifTrue:ifFalse:`\), some of the logical operation messages to Booleans \(`and:` and `or:`\), and the conditional repetition messages to blocks \(`whileTrue:` and `whileFalse:`\). The jump bytecodes indicate the next bytecode to be executed relative to the position of the jump. In other words, they tell the interpreter how many bytecodes to skip. 

% The following method for Rectangle includesPoint: uses a conditional jump.
% includesPoint: aPoint
% origin <= aPoint
% ifTrue: [^aPoint < corner]
% ifFalse: [^false]
% Rectangle includesPoint:	
% 0	push the value of the receiver's first instance variable (origin) onto the stack
% 16	push the contents of the first temporary frame location (the argument aPoint) onto the stack 
% 180	send a binary message with the selector <=
% 155	jump ahead 4 bytecodes if the object on top of the stack is false
% 16	push the contents of the first temporary frame location (the argument aPoint) onto the stack 
% 1	push the value of the receiver's second instance variable (corner) onto the stack 
% 178	send a binary message with the selector <
% 124	return the object on top of the stack as the value of the message ( includesPoint:) 
% 122	return false as the value of the message (includesPoint:) 












### The Interpreter


The virtual machine contains one interpreter and a compiler \(that compiles to assembly code\). Here we talk about the interpreter. It executes the bytecode instructions found in CompiledMethods. The interpreter uses five pieces of information refered hereafter as the state of the interpreter and repeatedly performs a three-step cycle.

#### The State of the Interpreter.

- The CompiledMethod whose bytecodes are being executed.
- The location of the next bytecode to be executed in that CompiledMethod. This is the interpreter's _instruction pointer_.
- The receiver and arguments of the message that invoked the CompiledMethod.
- Any temporary variables needed by the CompiledMethod.
- A stack.


The execution of most bytecodes involves the interpreter's stack. Push bytecodes tell where to find objects to add to the stack. Store bytecodes tell where to put objects found on the stack. Send bytecodes remove the receiver and arguments of messages from the stack. When the result of a message is computed, it is pushed onto the stack.

#### The Cycle of the Interpreter. 

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



### Contexts


Push, store, and jump bytecodes require only small changes to the state of the interpreter. 
Objects may be moved to or from the stack, and the instruction pointer is always changed; but most of the state remains the same. Send and return bytecodes require much larger changes to the interpreter's state. 

When a new message is sent \(for example message `x` or `-` above\), all five parts of the interpreter's state may have to be changed to execute a different CompiledMethod in response to this new message. The interpreter's old state must be remembered because the bytecodes after the send must be executed after the value of the message is returned. 

The interpreter saves its state in objects called _contexts_. There will be many contexts in the system at any one time. The context that represents the current state of the interpreter is called the _active_ context. When a send bytecode in the active context's CompiledMethod requires a new compiled method to be executed, the active context becomes _suspended_ and a new context is created and made active. The suspended context retains the state associated with the original compiled method until that context becomes active again. A context must remember the context that it suspended so that the suspended context can be resumed when a result is returned. The suspended context is called the new context's _sender_. 



We extend the form used to show the interpreter's state. The active context will be indicated by the word **Active** in its top delimiter. Suspended contexts will say **Suspended**. 


### Example


For example, consider a context representing the execution of the compiled method `Rectangle>>#rightCenter` with a receiver of `50@50  corner: 200@200`. The sender is some other context in the system. The source method is:

```
Rectangle >> rightCenter
	^self right @ self center y
```



#### After the execution of the first bytecode

The following active context shows that state of the context after the execution of the first bytecode:

**Active** Rectangle >> #rightCenter
- <4C> self - push the receiver \(self\) onto the stack
- **==>** <80> send: right - send a unary message with the selector in the first literal \(right\)
- <4C> self push the receiver \(self\) onto the stack
- <81> send: center  - send a unary message with the selector in the first literal \(center\)
- <7F> send: y  - send a unary message with the selector in the first literal \(y\)
- <6B> send: @  - send a unary message with the selector in the first literal \(@\)
- <5C> returnTop - return the object on top of the stack as the value of the message \(rightCenter\)

Literal frame
- 1 `#right` 
- 2 `#center`
- **Receiver:** 50@50  corner: 200@200
- **Arguments:**
- **Temporary Variables:**
- **Stack** 50@50  corner: 200@200



#### Execution of the second bytecode


After the next bytecode is executed, that context will be suspended. The object pushed by the first bytecode has been removed to be used as the receiver of a new context, which becomes active. The new active context is shown above the suspended context for the method `right`:

```
Rectangle >> right
	"Answer the position of the receiver's right vertical line."
	^ corner x 
```



**Active** Rectangle >> #right
- **==>**<01> pushRcvr: 1 - push the value of the receiver's second instance variable \(corner\) onto the stack
- <7E> send: x 
- <5C> returnTop

Literal frame
- **Receiver:** 50@50  corner: 200@200
- **Arguments:**
- **Temporary Variables:**
- **Stack:**



**Suspended** Rectangle >> #rightCenter
- <4C> self - push the receiver \(self\) onto the stack
- <80> send: right - send a unary message with the selector in the first literal \(right\)
- **==>** <4C> self push the receiver \(self\) onto the stack
- <81> send: center  - send a unary message with the selector in the first literal \(center\)
- <7F> send: y  - send a unary message with the selector in the first literal \(y\)
- <6B> send: @  - send a unary message with the selector in the first literal \(@\)
- <5C> returnTop - return the object on top of the stack as the value of the message \(rightCenter\)

Literal frame
- 1 `#right` 
- 2 `#center`
- **Receiver:** 50@50  corner: 200@200
- **Arguments:**
- **Temporary Variables:**
- **Stack:**



#### Execution of the right message


The next cycle of the interpreter advances the new context instead of the previous one. 

**Active** Rectangle >> #right
- <01> pushRcvr: 1 - push the value of the receiver's second instance variable \(corner\) onto the stack
- **==>**<7E> send: x 
- <5C> returnTop

Literal frame
- **Receiver:** 50@50  corner: 200@200
- **Arguments:**
- **Temporary Variables:**
- **Stack:** 200@200



**Suspended** Rectangle >> #rightCenter
- <4C> self - push the receiver \(self\) onto the stack
- <80> send: right - send a unary message with the selector in the first literal \(right\)
- **==>** <4C> self push the receiver \(self\) onto the stack
- <81> send: center  - send a unary message with the selector in the first literal \(center\)
- <7F> send: y  - send a unary message with the selector in the first literal \(y\)
- <6B> send: @  - send a unary message with the selector in the first literal \(@\)
- <5C> returnTop - return the object on top of the stack as the value of the message \(rightCenter\)

Literal frame
- 1 `#right` 
- 2 `#center`
- **Receiver:** 50@50  corner: 200@200
- **Arguments:**
- **Temporary Variables:**
- **Stack:**


#### Execution of the right message

In the next cycle, another message is sent, perhaps creating another context. Instead of following the response of this new message \(`x`\), we will skip to the point that this context returns a value \(to `right`\). When the result of `x` has been returned, the new context looks like this: 

**Active** Rectangle >> #right
- <01> pushRcvr: 1 - push the value of the receiver's second instance variable \(corner\) onto the stack
- <7E> send: x 
- **==>** <5C> returnTop

Literal frame
- **Receiver:** 50@50  corner: 200@200
- **Arguments:**
- **Temporary Variables:**
- **Stack:** 200



**Suspended** Rectangle >> #rightCenter
- <4C> self - push the receiver \(self\) onto the stack
- <80> send: right - send a unary message with the selector in the first literal \(right\)
- **==>** <4C> self push the receiver \(self\) onto the stack
- <81> send: center  - send a unary message with the selector in the first literal \(center\)
- <7F> send: y  - send a unary message with the selector in the first literal \(y\)
- <6B> send: @  - send a unary message with the selector in the first literal \(@\)
- <5C> returnTop - return the object on top of the stack as the value of the message \(rightCenter\)

Literal frame
- 1 `#right` 
- 2 `#center`
- **Receiver:** 50@50  corner: 200@200
- **Arguments:**
- **Temporary Variables:**
- **Stack:**




#### Returning from right


The next bytecode returns the value \(200\) on the top of the active context's stack as the value of the message that created the context \(`right`\). The active context's sender becomes the active context again and the returned value is pushed on its stack. It is ready to continue its execution.


**Active** Rectangle >> #rightCenter
- <4C> self - push the receiver \(self\) onto the stack
- <80> send: right - send a unary message with the selector in the first literal \(right\)
- **==>** <4C> self push the receiver \(self\) onto the stack
- <81> send: center  - send a unary message with the selector in the first literal \(center\)
- <7F> send: y  - send a unary message with the selector in the first literal \(y\)
- <6B> send: @  - send a unary message with the selector in the first literal \(@\)
- <5C> returnTop - return the object on top of the stack as the value of the message \(rightCenter\)

Literal frame
- 1 `#right` 
- 2 `#center`
- **Receiver:** 50@50  corner: 200@200
- **Arguments:**
- **Temporary Variables:**
- **Stack:** 200



### Handling blocks: inlined ones


```
Rectangle >> containsPoint: aPoint 
	"Answer whether aPoint is within the receiver."
	^origin <= aPoint and: [aPoint < corner]
```


- <00> pushRcvr: 0 
- <40> pushTemp: 0 
- <64> send: <= 
- <C3> jumpFalse: 41 
- <40> pushTemp: 0 
- <01> pushRcvr: 1 
- <62> send: < 
- <B0> jumpTo: 42 
- 41 <4E> pushConstant: false 
- 42 <5C> returnTop\)"



### Blocks


Context should have a receiver because executing `[ self foo ] value` should execute foo on self and not the block



### Primitive Methods?


The interpreter's actions after finding a compiled method depend on whether or not the compiled method indicates that a primitive method may be able to respond to the message. If no primitive method is indicated, a new method context is created and made active as described in previous sections. If a primitive method is indicated in the compiled method, the interpreter may be able to respond to the message without actually executing the bytecodes. For example, one of the primitive methods is associated with the `+` message to instances of `SmallInteger`.

```
SmallInteger >> + addend
	<primitive: 1>
	^super + addend
```


- <F8 01 00> callPrimitive: 1 
- <4C> self - push the receiver on the stack
- <40> pushTemp: 0 - push the first argument
- <EB 01> superSend: `+` - send a message via super
- <5C> returnTop  - return the object on top of the stack as the value of the message
